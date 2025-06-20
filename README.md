
[HttpPost]
public IActionResult GeneratePDFReport([FromBody] ExportWarehouseContainersReq req)
{
    var vm = new ExportWarehouseContainersViewModel
    {
        Req = req,
        WarehouseContainersList = new BLL.BLL().GetWarehouseContainers(req)
    };

    var pdfBytes = new BLL.BLL().GenerateWarehouseContainersReport(vm);
    return File(pdfBytes, "application/pdf", "WarehouseReport.pdf");
}
