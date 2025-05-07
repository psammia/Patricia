public class CustomerOrder
{
    public int CustomerOrderId { get; set; }

    public int CustomerId { get; set; }
    public int OrderId { get; set; }

    public bool IsPaid { get; set; }
    public decimal Amount { get; set; }
    public int NoOfProducts { get; set; }

    public Customer Customer { get; set; }
    public Order Order { get; set; }
}





CONTROLLER
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create([Bind("CustomerId,OrderId,IsPaid,Amount,NoOfProducts")] CustomerOrder customerOrder)
{
    if (ModelState.IsValid)
    {
        _context.Add(customerOrder);
        await _context.SaveChangesAsync();
        return RedirectToAction("Index");
    }

    ViewBag.Customers = new SelectList(_context.Customers, "CustomerId", "Name", customerOrder.CustomerId);
    ViewBag.Orders = new SelectList(_context.Orders, "OrderId", "OrderId", customerOrder.OrderId);
    return View(customerOrder);
}


CREATE VIEW
@model YourNamespace.Models.CustomerOrder

@{
    ViewBag.Title = "Add Customer to Order";
}

<h2>Add Customer to Order</h2>

@using (Html.BeginForm())
{
    @Html.AntiForgeryToken()
    
    <div class="form-horizontal">
        <div class="form-group">
            @Html.LabelFor(model => model.CustomerId, new { @class = "control-label col-md-2" })
            <div class="col-md-10">
                @Html.DropDownList("CustomerId", (SelectList)ViewBag.Customers, "Select Customer", new { @class = "form-control" })
            </div>
        </div>

        <div class="form-group">
            @Html.LabelFor(model => model.OrderId, new { @class = "control-label col-md-2" })
            <div class="col-md-10">
                @Html.DropDownList("OrderId", (SelectList)ViewBag.Orders, "Select Order", new { @class = "form-control" })
            </div>
        </div>

        <div class="form-group">
            @Html.LabelFor(model => model.Amount, new { @class = "control-label col-md-2" })
            <div class="col-md-10">
                @Html.TextBoxFor(model => model.Amount, new { @class = "form-control", type = "number", step = "0.01", min = "0" })
            </div>
        </div>

        <div class="form-group">
            @Html.LabelFor(model => model.NoOfProducts, new { @class = "control-label col-md-2" })
            <div class="col-md-10">
                @Html.TextBoxFor(model => model.NoOfProducts, new { @class = "form-control", type = "number", min = "1" })
            </div>
        </div>

        <div class="form-group">
            @Html.LabelFor(model => model.IsPaid, new { @class = "control-label col-md-2" })
            <div class="col-md-10">
                <div class="checkbox">
                    @Html.CheckBoxFor(model => model.IsPaid)
                </div>
            </div>
        </div>

        <div class="form-group">
            <div class="col-md-offset-2 col-md-10">
                <input type="submit" value="Add" class="btn btn-primary" />
            </div>
        </div>
    </div>
}


INDEX VIEW
@model IEnumerable<YourNamespace.Models.CustomerOrder>

<h2>Customer Orders</h2>

<table class="table table-bordered">
    <thead>
        <tr>
            <th>Customer</th>
            <th>Order</th>
            <th>Amount</th>
            <th>No of Products</th>
            <th>Is Paid</th>
        </tr>
    </thead>
    <tbody>
    @foreach (var item in Model)
    {
        <tr>
            <td>@item.Customer?.Name</td>
            <td>@item.Order?.OrderId</td>
            <td>@item.Amount.ToString("F2")</td>
            <td>@item.NoOfProducts</td>
            <td>@(item.IsPaid ? "Yes" : "No")</td>
        </tr>
    }
    </tbody>
</table>


