
I have 2 programs .net8.0 linked together to an API call:
1) GeneratePDF:
    in which i have these 2 functions:
   BaseController:
       [HttpPost]
    [Route("ExportWarehouseContainers")]
    public string ExportWarehouseContainers(ExportWarehouseContainersViewModel exportWarehouseContainersViewModel)
using Alterna.Archive.Core.Global;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.CookiePolicy;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Options;
using Newtonsoft.Json.Serialization;

internal class Program
{
    private static void Main(String[] args)
{
        BLL.BLL bll = new BLL.BLL();
        byte[] result = bll.GetByteArrayForWarehouseReport(
            exportWarehouseContainersViewModel.Req,
            exportWarehouseContainersViewModel.WarehouseContainersList);
        String? isRin = System.Configuration.ConfigurationManager.AppSettings["isRinActive"];

        StringBuilder sb = new StringBuilder(result.Length * 2);
        foreach (var b in result)
        {
            sb.AppendFormat("{0:x2}", b);
        }

        return sb.ToString();
    }
        WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

    BLL:
       public byte[] GetByteArrayForWarehouseReport(
    ExportWarehouseContainersReq req,
    List<ExportWareouseContainersDto> warehouseContainersList)
    {
        Settings.License = LicenseType.Community;

        byte[]? retRes = Document.Create(container =>
        builder.Services.AddControllers().AddNewtonsoftJson(options =>
   {
            container
           .Page(page =>
           {
               page.Margin(20);
               page.Size(PageSizes.A4);
               page.Header().Element(container => ComposeWarehouseReportHeader(container, req));
               page.Content().Element(container => ComposeWarehouseReportContent(container, warehouseContainersList));
               //page.Footer().Element(container => ComposeWarehouseReportFooter(container));
           });
        }).GeneratePdf();

        byte[] empty = [];

        DynamicParameters dynamicParameters = new();
        dynamicParameters.Add("PDF", empty, DbType.Binary, ParameterDirection.Input);
        dynamicParameters.Add("Request", JsonConvert.SerializeObject(req, Formatting.Indented),
            DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("ApiMethod", "ExportWarehouseContainers", DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("BranchList", req.BranchList, DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("Entity", req.Entity, DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("User", req.User, DbType.String, ParameterDirection.Input);

        using (DAL.DAL dal = new(Catalog_Archive, out var res))
            options.SerializerSettings.ContractResolver = new DefaultContractResolver();
        });
        builder.Services.AddControllers().AddJsonOptions(options =>
        {
            options.JsonSerializerOptions.PropertyNamingPolicy = null;
        });
        // Add services to the container.
        if (isRin == "true")
   {
            var command = ConfigurationManager.AppSettings["Insert_PDF_SP"] ?? "usp_InsertPDF";
            dal.ExecuteQuery(command, dynamicParameters);
            builder.Logging.AddRinLogger();
            builder.Services.AddRin();
            builder.Services.AddControllersWithViews().AddRinMvcSupport();
   }

        return retRes;
    }

   The other program: Archiving: which call the ExportWarehouseContainers:
   I have the view in which i get the list of containers in warehouse, and in it i have a button that generate a report in pdf, using the data i got already from my copntroller:
   WarehouseController:
           public ActionResult GetWarehouseContainer(WarehouseContainerSearchModel searchModel)
        else
   {
            WarehouseContainerTableModel tableModel = new();

            GetWarehouseContainersReq ApiReq = new()
            {
                BaseReq = new BaseRequest(HttpContext,GetSession("ArchiveData")),
            };

            if (!String.IsNullOrWhiteSpace(searchModel.CompanyCode))
            {
                ApiReq.CompanyCode = searchModel.CompanyCode;
            }
            if (!String.IsNullOrWhiteSpace(searchModel.Code))
            {
                ApiReq.Code = searchModel.Code;
            }
            if (!String.IsNullOrWhiteSpace(searchModel.FromDate))
            {
                ApiReq.FromDate = Common.FormatDate(searchModel.FromDate);
            }
            if (!String.IsNullOrWhiteSpace(searchModel.ToDate))
            {
                ApiReq.ToDate = Common.FormatDate(searchModel.ToDate);
            }
            if (!String.IsNullOrWhiteSpace(searchModel.StatusCode))
            {
                ApiReq.StatusCode = searchModel.StatusCode;
            }

            GetWarehouseContainersRes ApiResp = Common.ApiCall<GetWarehouseContainersRes>(ApiReq, "GetWarehouseContainers");

            tableModel.WarehouseContainerList = ApiResp.Resp;

            return PartialView("_WarehouseSearchContainerTable", tableModel);
            builder.Services.AddControllersWithViews();
   }

   API: Back end controller: ArchivingController
           #region GetWarehouseContainers Controller
        [HttpPost]
        [Route("GetWarehouseContainers")]
        public GetWarehouseContainersRes GetWarehouseContainers(GetWarehouseContainersReq getWarehouseContainersReq)
        builder.Services.AddDistributedMemoryCache();
        builder.Services.AddSession();
#if !DEBUG
        if (!String.IsNullOrEmpty(System.Configuration.ConfigurationManager.AppSettings["DomainConfig"]))
{
            GetWarehouseContainersRes response = new()
            {
                Req = getWarehouseContainersReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getWarehouseContainersReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetWarehouseContainers",
                UserName = getWarehouseContainersReq.BaseReq.CurrentUser
            };

            try
            builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie(options =>
{
                String CorrelationId = String.IsNullOrEmpty(getWarehouseContainersReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getWarehouseContainersReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getWarehouseContainersReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getWarehouseContainersReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getWarehouseContainersReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getWarehouseContainersReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getWarehouseContainersReq.BaseReq.CurrentEntity)} and {nameof(getWarehouseContainersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getWarehouseContainersReq.BaseReq.CurrentEntity) ? String.Empty : getWarehouseContainersReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getWarehouseContainersReq.BaseReq.CurrentBranch) ? String.Empty : getWarehouseContainersReq.BaseReq.CurrentBranch;

                LogInfo("GetWarehouseContainers Has been called with the following Request", correlationInfo);
                LogInfoJson(getWarehouseContainersReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getWarehouseContainersReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetWarehouseContainers call", correlationInfo);

                    response.Resp = oBLL.GetWarehouseContainers(getWarehouseContainersReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Container have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetWarehouseContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetWarehouseContainers is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getWarehouseContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getWarehouseContainersReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getWarehouseContainersReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getWarehouseContainersReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getWarehouseContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getWarehouseContainersReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getWarehouseContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getWarehouseContainersReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
                options.Cookie.Domain = System.Configuration.ConfigurationManager.AppSettings["DomainConfig"];
                options.Cookie.SameSite = SameSiteMode.Lax;
                options.Cookie.SecurePolicy = CookieSecurePolicy.SameAsRequest;
                options.Cookie.HttpOnly = true;
                options.Cookie.Path = "/";
            });
}
        #endregion
#endif

in BLL.cs i have: 
        #region GetWarehouseContainers

        public List<Container> GetWarehouseContainers(GetWarehouseContainersReq getWarehouseContainersReq)
        builder.Services.AddMvcCore().AddRazorViewEngine(opt =>
   {
            DAL.DAL iDAL = new();
            List<Container> RetList = [];

            OnPreEventGetWarehouseContainers?.Invoke(ref getWarehouseContainersReq);

            DynamicParameters param = new();

            param.Add("FromDate", getWarehouseContainersReq.FromDate);
            param.Add("ToDate", getWarehouseContainersReq.ToDate);
            param.Add("Code", getWarehouseContainersReq.Code);
            param.Add("CompanyCode", getWarehouseContainersReq.CompanyCode);
            param.Add("StatusCode", getWarehouseContainersReq.StatusCode);

            RetList = iDAL.ExecuteQuery<Container>("usp_GetWarehouseContainers", param, CommandType.StoredProcedure,
                CommandDirection.Select);

            opt.ViewLocationFormats.Add("/Views/{1}/{0}.cshtml");
            opt.ViewLocationFormats.Add("/Views/{1}/PartialViews/{0}.cshtml");
            opt.ViewLocationFormats.Add("/Views/Shared/{0}.cshtml");
        });

            OnPostEventGetWarehouseContainers?.Invoke(ref RetList, ref getWarehouseContainersReq);

            return RetList;
        }
        WebApplication app = builder.Build();

in customer.cs i have:
        #region Generate Warehouse Containers Report
        public byte[] GenerateWarehouseContainersReport(ExportWarehouseContainersViewModel vm)
        // Configure the HTTP request pipeline.
        if (!app.Environment.IsDevelopment())
   {
            String data = JsonConvert.SerializeObject(vm);
            HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
            HttpClient client = new();
            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ??
                                    throw new SGBLInternalServerException(
                                        "PDF Service not initialized please Contact Support");

            Task<HttpResponseMessage>
                Request = client.PostAsync($"{PDFRequestBase}ExportWarehouseContainers", content);

            Request.Wait();
            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
            responseString.Wait();

            string result = responseString.Result;

            Byte[] bytearray = new Byte[result.Length / 2];
            for (Int32 i = 0; i < result.Length; i += 2)
            {
                bytearray[i / 2] = Convert.ToByte(result.Substring(i, 2), 16);
            }
            return bytearray;
            app.UseExceptionHandler("/Home/Error");
   }
        #endregion
    }


    main view: Warehouse/ Search
    @using Alterna.Archive.Core.Models.SearchModel
@model Alterna.Archive.Core.Models.SearchModel.WarehouseContainerSearchModel
@{
    ViewBag.Title = "Warehouse Search Boxes";
}

<div class="row">
    <div class="col-md-12">
        <h3>Warehouse Management</h3>
    </div>
    <div class="col-md-12">
        <ol class="breadcrumb sgbl-breadcrumb">
            <li><a href="~/Home/Index/Redirect">Home</a></li>
            <li class="active">Search Box</li>
        </ol>
    </div>
</div>

<div class="card">
    <div class="card-header"></div>
    <div class="card-content collapse show">
        <div class="card-body card-dashboard">
            <div class="row" id="WarehouseContainersFilterOptions">

                @{
                    var currentFormattedDate = "Today " + @DateTime.Now.ToString("dd MMMM, yyyy");
                }

                <div class="form-group col-md-4">
                    <label>From Date </label>
                    <input id="dpFromDate" class="datepicker form-control" placeholder="Select A Date" />
                </div>

                <div class="form-group col-md-4">
                    <label>To Date </label>
                    <input id="dpToDate" class="datepicker form-control" placeholder="Select A Date" />
                </div>

                <div class="form-group col-md-4">
                    <label>Box Ref </label>
                    <input id="IContainerCode" type="text" class="form-control">
                </div>

                <div class="form-group col-md-4">
                    <label>Branch </label>
                    <select class="form-control selectpicker" id="SCompanyCode" data-live-search="true">
                        <option value="">No Branch Selected</option>
                        @{
                            foreach ((String companyCode, String companyName) in Model.CompaniesDict)
                            {
                                <option value="@companyCode">@companyCode - @companyName</option>
                            }
                        }
                    </select>
                </div>

                <div class="form-group col-md-4">
                    <label>Box Status</label>
                    <select class="form-control selectpicker" id="StatusCode" data-live-search="true">
                        <option value="">All</option>
                        <option value="SENT">SENT</option>
                        <option value="RECEIVED">RECEIVED</option>
                        <option value="TOBEDESTR">TOBEDESTR</option>
                        <option value="DESTROYED">DESTROYED</option>
                    </select>
                </div>

                <div class="form-group col-md-4">
                </div>

                <div class="col-md-4">
                    <div>&#8291;</div>
                    <button type="button" class="btn btn-primary" style="margin-top: 6px" onclick="showData()">Search</button>
                </div>

            </div>

            <div id="TableDisplay" class="table-spacer"></div>
        </div>
    </div>
</div>

<script>
    $('.datepicker').pickadate({
        labelMonthNext: 'Go to the next month',
        labelMonthPrev: 'Go to the previous month',
        labelMonthSelect: 'Pick a month from the dropdown',
        labelYearSelect: 'Pick a year from the dropdown',
        selectMonths: true,
        selectYears: 20,
        closeOnClear: false,
        firstDay: 1
    });

    function showData() {

        $.ajax({
            type: 'POST',
            url: '/Warehouse/GetWarehouseContainer/',
            data: {
                searchModel:
                {
                    CompanyCode: $('#SCompanyCode').find(":selected").val(),
                    Code: $('#IContainerCode').val(),
                    FromDate: $('#dpFromDate').val(),
                    ToDate: $('#dpToDate').val(),
                    StatusCode: $('#StatusCode').val()
                }
            },
            dataType: 'html',
            success: function (response) {
                if (!ValidationBetweenDates()) {
                    return;
                }
                $("#WarehouseContainersFilterOptions").hide();
                $('#TableDisplay').html(response);
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
        return false;
    }

    function ValidationBetweenDates() {
        var from = $("#dpFromDate").val();
        var to = $("#dpToDate").val();

        if (Date.parse(from) > Date.parse(to)) {
            swal("Invalid Date Range", "End Date must be greater than Start Date", "error");
            return false;
        }
        else {
            return true;
        else
        {
            app.UseDeveloperExceptionPage();
   }
    }
</script>


partial View: _WarehouseSearchContainerTable
@using Alterna.Archive.Core.Models
@model Alterna.Archive.Core.Models.TableModel.WarehouseContainerTableModel


<table id="TblWarehouseReceiveContainerTable" class="table table-striped table-bordered" style="width:100%;">
    <thead>
        <tr>
            <th>Box Ref</th>
            <th>Branch</th>
            <th>Status</th>
            <th>Archiving Date</th>
            <th>Destruction Date</th>
            <th>Archiving Period</th>
            <th>Sent By</th>
            <th>Received By</th>
            <th>Received Date</th>

        </tr>
    </thead>
    <tbody>
        @if (Model.WarehouseContainerList.Count > 0)
        if (isRin == "true")
   {
            foreach (Container container in Model.WarehouseContainerList)
            {
                var iconId = "containerDetails" + container.Id;
                <tr>
                    <td>@container.Code</td>
                    <td>@container.CompanyCode</td>
                    <td>@container.StatusCode</td>
                    @{
                        if (@container.ArchivingDate.HasValue)
                        {
                            <td>@container.ArchivingDate.Value.ToString("dd/MM/yyyy")</td>

                            if (container.DestructionDate.HasValue)
                            {
                                <td>@container.DestructionDate.Value.ToString("dd/MM/yyyy")</td>
                            }else
                            {
                                <td></td>
                            }
            app.UseRin();

                            if (container.ArchivingPeriod == -1)
                            {
                                <td>Unlimited</td>

                            }
                            else
                            {
                                <td>@container.ArchivingPeriod</td>
                            }
                        }
                        else
                        {
                            <td></td>
                            <td></td>
                            <td></td>
                        }
                    }

                    <td>@container.SentBy</td>
                    <td>@container.ReceivedBy</td>

                    @if (container.ReceivedDate.HasValue)
                    {
                        <td>@container.ReceivedDate.Value.ToString("dd/MM/yyyy")</td>
                    }
                    else
                    {
                        <td></td>
                    }

                </tr>
            }
            app.UseRinDiagnosticsHandler();
   }
    </tbody>
</table>

<button id="BtnSearchAgain" type="button" class="btn btn-primary" style="margin-top: 6px" onclick="SearchAgain()">Search Again</button>
<button id="BtnGenerateReport" type="button" class="btn btn-success" style="margin-top: 6px" onclick="GenerateReport()">Generate Report</button>

<script>
    $(document).ready(() => {
        $("#TblWarehouseReceiveContainerTable").DataTable(
            {
                pagingType: 'full_numbers',
                "scrollX": true,
                order: [[0, 'desc']]
            });
    })
        app.UseStaticFiles();

        app.UseRouting();

    function SearchAgain() {
        $('#TableDisplay').html("");;
        $("#WarehouseContainersFilterOptions").show();
        app.UseAuthorization();
        app.UseCookiePolicy();
        app.UseSession();
        app.MapControllerRoute(
            name: "default",
            pattern: "{controller=Home}/{action=Index}/{Param1?}/{Param2?}");
        app.UseStatusCodePagesWithReExecute("/Home/Error");
        app.Run();
}

    function GenerateReport() {
        $.ajax({
            type: 'POST',
            url: '/Warehouse/GetWarehouseContainer/',
            data: {},
            dataType: 'html',
            success: function (response) {
                $('#TableDisplay').html(response);
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });


</script>

My problem, is that i run first the generatepdf, and that the archiving program, when clicking on generate report in the partial view, i got this medssage from the inspect/console:
Uncaught SyntaxError: Failed to execute 'appendChild' on 'Node': Unexpected end of input
    at b (jquery.min.js:2:866)
    at He (jquery.min.js:2:48373)
    at S.append (jquery.min.js:2:49724)
    at S.<anonymous> (jquery.min.js:2:50816)
    at $ (jquery.min.js:2:32425)
    at S.html (jquery.min.js:2:50494)
    at Object.success (Search:544:36)
    at c (jquery.min.js:2:28327)
    at Object.fireWith [as resolveWith] (jquery.min.js:2:29072)
    at l (jquery.min.js:2:79901)Understand this error
Search:1 Uncaught ReferenceError: GenerateReport is not defined
    at HTMLButtonElement.onclick (Search:1:1)

    show me where is my mistake, missing syntax, or any correction i have to do
        
   
   
   
   
}
