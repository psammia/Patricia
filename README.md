// CONTINUATION: Add Excel Export and Cash Flow Summary

// In OrdersController.cs public ActionResult ExportToExcel() { var orders = _orderRepo.GetOrders(); using (var package = new OfficeOpenXml.ExcelPackage()) { var worksheet = package.Workbook.Worksheets.Add("Orders"); worksheet.Cells[1, 1].Value = "Customer"; worksheet.Cells[1, 2].Value = "Total Cost"; worksheet.Cells[1, 3].Value = "Total Paid"; worksheet.Cells[1, 4].Value = "Profit"; worksheet.Cells[1, 5].Value = "Status";

int row = 2;
    foreach (var o in orders)
    {
        worksheet.Cells[row, 1].Value = o.CustomerName;
        worksheet.Cells[row, 2].Value = o.TotalCost;
        worksheet.Cells[row, 3].Value = o.TotalPaid;
        worksheet.Cells[row, 4].Value = o.Profit;
        worksheet.Cells[row, 5].Value = o.Status;
        row++;
    }

    var stream = new System.IO.MemoryStream();
    package.SaveAs(stream);
    stream.Position = 0;
    return File(stream, "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", "Orders.xlsx");
}

}

// Add Cash Flow Summary View (CashFlow.cshtml) @model dynamic

<h2>Cash Flow Summary</h2>
<p>Total Orders: @Model.TotalOrders</p>
<p>Total Cost: @Model.TotalCost</p>
<p>Total Paid: @Model.TotalPaid</p>
<p>Total Profit: @Model.TotalProfit</p>// In OrdersController.cs public ActionResult CashFlow() { var orders = _orderRepo.GetOrders(); var summary = new { TotalOrders = orders.Count, TotalCost = orders.Sum(o => o.TotalCost), TotalPaid = orders.Sum(o => o.TotalPaid), TotalProfit = orders.Sum(o => o.Profit) }; return View(summary); }

// Add links to Index.cshtml @Html.ActionLink("Export to Excel", "ExportToExcel") | @Html.ActionLink("Cash Flow Summary", "CashFlow")


