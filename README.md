public class OrderSummaryViewModel {
    public int OrderId { get; set; }
    public string CustomerName { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal TotalCost { get; set; }
    public decimal TotalPrice { get; set; }
    public decimal Profit => TotalPrice - TotalCost;
    public bool IsPaid { get; set; }
    public DateTime? PaidDate { get; set; }
}


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



public class CustomerRepository
{
    private readonly DatabaseHelper _db;

    public CustomerRepository(DatabaseHelper db)
    {
        _db = db;
    }

    public List<Customer> GetAll()
    {
        var customers = new List<Customer>();
        using var conn = _db.GetConnection();
        conn.Open();
        using var cmd = new SqlCommand("SELECT * FROM Customers", conn);
        using var reader = cmd.ExecuteReader();
        while (reader.Read())
        {
            customers.Add(new Customer
            {
                CustomerId = (int)reader["CustomerId"],
                Name = reader["Name"].ToString(),
                Email = reader["Email"].ToString(),
                Phone = reader["Phone"].ToString()
            });
        }
        return customers;
    }

    public void Add(Customer customer)
    {
        using var conn = _db.GetConnection();
        conn.Open();
        using var cmd = new SqlCommand("INSERT INTO Customers (Name, Email, Phone) VALUES (@Name, @Email, @Phone)", conn);
        cmd.Parameters.AddWithValue("@Name", customer.Name);
        cmd.Parameters.AddWithValue("@Email", customer.Email ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@Phone", customer.Phone ?? (object)DBNull.Value);
        cmd.ExecuteNonQuery();
    }
}
