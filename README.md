// ===========================
// 1. VIEW (Search.cshtml)
// ===========================
<script>
    function GenerateReport() {
        var request = {
            BaseReq: {
                CurrentBranch: $('#SCompanyCode').val(),
                CurrentEntity: 'YourEntityHere', // Replace with actual entity
                CurrentUser: 'YourUsernameHere'   // Replace with actual username
            },
            FromDate: $('#dpFromDate').val(),
            ToDate: $('#dpToDate').val(),
            Code: $('#IContainerCode').val(),
            CompanyCode: $('#SCompanyCode').val(),
            StatusCode: $('#StatusCode').val()
        };

        $.ajax({
            url: '/Warehouse/GeneratePDFReport',
            type: 'POST',
            data: JSON.stringify(request),
            contentType: 'application/json',
            xhrFields: { responseType: 'blob' },
            success: function (data, status, xhr) {
                var blob = new Blob([data], { type: "application/pdf" });
                var downloadUrl = URL.createObjectURL(blob);
                var a = document.createElement("a");
                a.href = downloadUrl;
                a.download = "WarehouseReport.pdf";
                document.body.appendChild(a);
                a.click();
                a.remove();
            },
            error: function (xhr) {
                alert("Failed to generate PDF. " + xhr.responseText);
            }
        });
    }
</script>

// ===========================
// 2. CONTROLLER METHOD
// ===========================
[HttpPost]
public IActionResult GeneratePDFReport([FromBody] ExportWarehouseContainersReq req)
{
    ExportWarehouseContainersViewModel vm = new ExportWarehouseContainersViewModel
    {
        Req = req,
        WarehouseContainersList = new BLL.BLL().GetWarehouseContainers(req)
    };

    byte[] pdfBytes = new BLL.BLL().GenerateWarehouseContainersReport(vm);

    return File(pdfBytes, "application/pdf", "WarehouseReport.pdf");
}

// ===========================
// 3. BLL Method to get containers
// ===========================
public List<Container> GetWarehouseContainers(ExportWarehouseContainersReq req)
{
    DAL.DAL iDAL = new DAL.DAL();
    DynamicParameters param = new();

    param.Add("FromDate", req.FromDate);
    param.Add("ToDate", req.ToDate);
    param.Add("Code", req.Code);
    param.Add("CompanyCode", req.CompanyCode);
    param.Add("StatusCode", req.StatusCode);

    return iDAL.ExecuteQuery<Container>("usp_GetWarehouseContainers", param, CommandType.StoredProcedure, CommandDirection.Select);
}

// ===========================
// 4. BLL Method to generate PDF
// ===========================
public byte[] GenerateWarehouseContainersReport(ExportWarehouseContainersViewModel vm)
{
    string json = JsonConvert.SerializeObject(vm);
    HttpContent content = new StringContent(json, Encoding.UTF8, "application/json");

    HttpClient client = new();
    string pdfServiceUrl = ConfigurationManager.AppSettings["PDFService"];
    if (string.IsNullOrWhiteSpace(pdfServiceUrl))
        throw new Exception("PDF Service URL is missing");

    var response = client.PostAsync($"{pdfServiceUrl}ExportWarehouseContainers", content).Result;
    var hexResult = response.Content.ReadAsStringAsync().Result;

    byte[] byteArray = new byte[hexResult.Length / 2];
    for (int i = 0; i < hexResult.Length; i += 2)
    {
        byteArray[i / 2] = Convert.ToByte(hexResult.Substring(i, 2), 16);
    }

    return byteArray;
}

// ===========================
// 5. ViewModel
// ===========================
public class ExportWarehouseContainersViewModel
{
    public ExportWarehouseContainersReq Req { get; set; }
    public List<Container> WarehouseContainersList { get; set; }
}

// ===========================
// 6. Request DTO
// ===========================
public class ExportWarehouseContainersReq
{
    public BaseRequest BaseReq { get; set; }
    public string FromDate { get; set; }
    public string ToDate { get; set; }
    public string Code { get; set; }
    public string CompanyCode { get; set; }
    public string StatusCode { get; set; }
}

public class BaseRequest
{
    public string CurrentUser { get; set; }
    public string CurrentEntity { get; set; }
    public string CurrentBranch { get; set; }
}
