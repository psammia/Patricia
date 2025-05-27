🔧 Key Fixes Required
✅ 1. Fix the POST Edit action
You're currently passing only int[] selectedCustomers, but your view posts a full list of CustomerOrders, which includes:

CustomerId

Amount

IsPaid

NoOfProductperCustomer

🔁 Update the Edit POST method to accept the full Order model (including CustomerOrders):

csharp
Copy
Edit
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(Order order)
{
    if (order.CustomerOrders == null || !order.CustomerOrders.Any())
    {
        ModelState.AddModelError("", "Please add at least one customer.");
    }

    if (!ModelState.IsValid)
    {
        ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
        ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
        return View(order);
    }

    await _repo.UpdateOrderWithCustomersAsync(order);
    return RedirectToAction("Index");
}
✅ 2. Fix UpdateOrderWithCustomersAsync in your repository
You're currently trying to accept just customerIds, but you actually need the full list of CustomerOrders (with all fields, not just IDs).

🔁 Replace method signature and logic:

csharp
Copy
Edit
public async Task UpdateOrderWithCustomersAsync(Order order)
{
    using var conn = Connection;
    conn.Open();
    using var tran = conn.BeginTransaction();

    try
    {
        await conn.ExecuteAsync("UPDATE Orders SET OrderDate = @OrderDate, Profit = @Profit, NoOfProduct=@NoOfProduct, TotalAmount=@TotalAmount, StatusCode=@StatusCode WHERE OrderId = @OrderId", order, tran);

        // Delete previous customer associations
        await conn.ExecuteAsync("DELETE FROM CustomerOrders WHERE OrderId = @OrderId", new { order.OrderId }, tran);

        // Reinsert updated customer data
        foreach (var co in order.CustomerOrders)
        {
            await conn.ExecuteAsync(
                "INSERT INTO CustomerOrders (OrderId, CustomerId, IsPaid, Amount, NoOfProductperCustomer) VALUES (@OrderId, @CustomerId, @IsPaid, @Amount, @NoOfProductperCustomer)",
                new
                {
                    OrderId = order.OrderId,
                    CustomerId = co.CustomerId,
                    Amount = co.Amount,
                    IsPaid = co.IsPaid,
                    NoOfProductperCustomer = co.NoOfProductperCustomer
                }, tran);
        }

        tran.Commit();
    }
    catch
    {
        tran.Rollback();
        throw;
    }
}
✅ 3. Fix the select list in Edit.cshtml
You're not selecting the existing customer properly in the dropdown.

🔁 In this block:

cshtml
Copy
Edit
<option value="@customer.CustomerId")>@customer.Name</option>
You're missing the selected attribute. Fix it like this:

cshtml
Copy
Edit
<option value="@customer.CustomerId" @(customer.CustomerId == co.CustomerId ? "selected" : "")>@customer.Name</option>
💡 Optional: Add Validation Client-Side
Ensure you have this line in _ValidationScriptsPartial.cshtml:

html
Copy
Edit
@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        // Your dynamic JS logic
    </script>
}
✅ Summary of Changes
Area	Fix
Edit POST in Controller	Accept full Order with CustomerOrders, not just customer IDs
UpdateOrderWithCustomersAsync	Use Order.CustomerOrders instead of int[]
Razor View Dropdown	Set selected attribute properly
Validation	Retain client-side validation script
