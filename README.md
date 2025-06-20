$('#notifyButton').on('click', function () {
    const selectedIds = $('.row-checkbox:checked').map(function () {
        return $(this).val();
    }).get();

    if (selectedIds.length === 0) return;

    const data = {
        containerIds: selectedIds.join(',')
    };

    $.ajax({
        type: 'POST',
        url: '/BoxEntity/NotifyWareHouseAndGeneratePdf/', // 👈 You create this new endpoint below
        data: data,
        xhrFields: { responseType: 'blob' },
        success: function (response, status, xhr) {
            // Download the PDF
            const blob = new Blob([response], { type: "application/pdf" });
            const link = document.createElement('a');
            link.href = window.URL.createObjectURL(blob);
            link.download = "WarehouseNotification.pdf";
            link.click();

            // Remove rows from DataTable
            $('.row-checkbox:checked').each(function () {
                const row = $(this).closest('tr');
                table.row(row).remove().draw();
            });

            // Disable button
            $('#notifyButton').prop('disabled', true);
        },
        error: function (xhr) {
            $('#MainRenderLocation').html(xhr.responseText);
        }
    });
});


[HttpPost]
public IActionResult NotifyWareHouseAndGeneratePdf(string containerIds)
{
    if (string.IsNullOrEmpty(containerIds))
        return BadRequest("No containers selected");

    var containerIdList = containerIds.Split(',').Select(int.Parse).ToList();

    // Step 1: Update status to "SENT" in DB
    _yourService.NotifyContainers(containerIdList); // ⬅️ Update status in your DB here

    // Step 2: Get containers info for PDF
    List<ExportWareouseContainersDto> containers = _yourService.GetContainerDetails(containerIdList);

    var req = new ExportWarehouseContainersReq
    {
        BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
        User = "CurrentUser",
        Entity = "CurrentEntity",
        BranchList = "BranchCode"
    };

    ExportWarehouseContainersViewModel vm = new()
    {
        Req = req,
        WarehouseContainersList = containers
    };

    var customer = new Customer(); // or injected service
    var pdfBytes = customer.GenerateWarehouseContainersReport(vm);

    return File(pdfBytes, "application/pdf", "WarehouseNotification.pdf");
}


