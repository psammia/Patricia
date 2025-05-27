Here is the complete corrected setup for your Order Creation View and the POST Create controller action, based on your models and SQL structure:

✅ Order.cs model
csharp
Copy
Edit
public class Order
{
    public int OrderId { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal Profit { get; set; }
    public decimal TotalAmount { get; set; }
    public int NoOfProduct { get; set; }
    public string StatusCode { get; set; }

    public List<CustomerOrder> CustomerOrders { get; set; } = new();
}
✅ CustomerOrder.cs model
csharp
Copy
Edit
public class CustomerOrder
{
    public int OrderId { get; set; }
    public int CustomerId { get; set; }
    public decimal Amount { get; set; }
    public int NoOfProduct { get; set; }
    public bool IsPaid { get; set; }

    // Optional helper
    [NotMapped]
    public string CustomerName { get; set; }
}
✅ Controller Create (POST)
csharp
Copy
Edit
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Create(Order order)
{
    if (ModelState.IsValid)
    {
        // Save Order and CustomerOrders using Dapper or EF
        // Example:
        // int newOrderId = SaveOrder(order);
        // foreach (var co in order.CustomerOrders)
        // {
        //     co.OrderId = newOrderId;
        //     SaveCustomerOrder(co);
        // }

        return RedirectToAction("Index");
    }

    ViewBag.Customers = _service.GetCustomers();    // Repopulate
    ViewBag.Statuses = _service.GetStatuses();
    return View(order);
}
✅ View Create.cshtml
cshtml
Copy
Edit
@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<Customer> ?? new();
    var customerOptionsHtml = new System.Text.StringBuilder();
    foreach (var customer in customers)
    {
        customerOptionsHtml.Append($"<option value=\"{customer.CustomerId}\">{customer.Name}</option>");
    }
}

<h2>Create Order</h2>

<form asp-action="Create" method="post">
    <div class="mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" type="date" value="@DateTime.Today.ToString("yyyy-MM-dd")" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="TotalAmount" class="form-label"></label>
        <input asp-for="TotalAmount" class="form-control" />
        <span asp-validation-for="TotalAmount" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Profit" class="form-label"></label>
        <input asp-for="Profit" class="form-control" />
        <span asp-validation-for="Profit" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="NoOfProduct" class="form-label"></label>
        <input asp-for="NoOfProduct" class="form-control" />
        <span asp-validation-for="NoOfProduct" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label>Status</label>
        <select asp-for="StatusCode" class="form-select">
            <option value="">Not Delivered</option>
            @foreach (var status in (IEnumerable<Status>)ViewBag.Statuses)
            {
                <option value="@status.StatusCode">@status.StatusCode</option>
            }
        </select>
        <span asp-validation-for="StatusCode" class="text-danger"></span>
    </div>

    <h4>Customers</h4>
    <div id="customerRepeater" class="mb-3"></div>
    <button type="button" id="addCustomer" class="btn btn-success mb-3">Add Customer</button>

    <div class="form-group">
        <input type="submit" value="Create" class="btn btn-primary" />
    </div>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = 0;
        const customerOptionsHtml = `@Html.Raw(customerOptionsHtml.ToString())`;

        function getCustomerEntryHtml(i) {
            return `
                <div class="customer-entry mb-3 border p-3 rounded">
                    <label>Customer</label>
                    <select class="form-select customer-select" name="CustomerOrders[${i}].CustomerId" required>
                        <option value="">-- Select Customer --</option>
                        ${customerOptionsHtml}
                    </select>

                    <label class="mt-2">Amount</label>
                    <input name="CustomerOrders[${i}].Amount" type="number" step="0.01" class="form-control" required />

                    <label class="mt-2">No. of Product</label>
                    <input name="CustomerOrders[${i}].NoOfProduct" type="number" class="form-control" required />

                    <div class="form-check mt-2">
                        <input type="hidden" name="CustomerOrders[${i}].IsPaid" value="false" />
                        <input name="CustomerOrders[${i}].IsPaid" type="checkbox" value="true" class="form-check-input" />
                        <label class="form-check-label">Is Paid</label>
                    </div>

                    <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
                </div>
            `;
        }

        $(document).ready(function () {
            $('#customerRepeater').append(getCustomerEntryHtml(index++));
            $('#addCustomer').on('click', function () {
                $('#customerRepeater').append(getCustomerEntryHtml(index++));
            });

            $(document).on('click', '.remove-entry', function () {
                $(this).closest('.customer-entry').remove();
                // Re-index names
                $('#customerRepeater .customer-entry').each(function (i, el) {
                    $(el).find('select, input').each(function () {
                        const oldName = $(this).attr('name');
                        if (oldName) {
                            const newName = oldName.replace(/CustomerOrders\[\d+\]/, `CustomerOrders[${i}]`);
                            $(this).attr('name', newName);
                        }
                    });
                });
                index = $('#customerRepeater .customer-entry').length;
            });
        });
    </script>
}
🧪 Testing Tips
✅ Make sure form POST hits the controller.

✅ Use ModelState.IsValid to validate required fields.

✅ Debug order.CustomerOrders in the controller to verify correct binding.

Let me know if you want a Dapper or EF Core insert logic for this structure.
