function getCustomerEntryHtml(i) {
    return `
        <div class="customer-entry mb-3 border p-3 rounded">
            <label>Customer</label>
            <select class="form-select customer-select" name="CustomerOrders[${i}].CustomerId" required>
                <option value="">-- Select Customer --</option>
                ${customerOptionsHtml}
            </select>

            <label class="mt-2">Amount</label>
            <input name="CustomerOrders[${i}].Amount" class="form-control" type="number" step="0.01" required />

            <label class="mt-2">No. of Product</label>
            <input name="CustomerOrders[${i}].NoOfProduct" class="form-control" type="number" required />

            <div class="form-check mt-2">
                <input type="hidden" name="CustomerOrders[${i}].IsPaid" value="false" />
                <input name="CustomerOrders[${i}].IsPaid" type="checkbox" value="true" class="form-check-input" />
                <label class="form-check-label">Is Paid</label>
            </div>

            <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
        </div>
    `;
}


✅ 4. Controller Method – Create (POST)
csharp
Copy
Edit
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Create(Order order)
{
    if (ModelState.IsValid)
    {
        // Check if CustomerOrders were bound correctly
        foreach (var co in order.CustomerOrders)
        {
            // Debug here if needed
            Console.WriteLine($"CustomerId: {co.CustomerId}, Amount: {co.Amount}, NoOfProduct: {co.NoOfProduct}, IsPaid: {co.IsPaid}");
        }

        // Save order and its customer orders to DB using Dapper or EF
        // Insert order first and get the generated OrderId
        // Then loop through customerOrders, set OrderId, and insert

        return RedirectToAction("Index");
    }

    // If model state invalid, repopulate dropdowns
    ViewBag.Customers = _yourService.GetCustomers();
    ViewBag.Statuses = _yourService.GetStatuses();

    return View(order);
}
✅ 5. Summary of Common Pitfalls
Issue	Cause	Fix
CustomerOrders[i].IsPaid is always false	Checkbox only posts if checked	Use hidden input before checkbox
CustomerOrders list is empty	Missing or wrong input name attributes	Ensure proper name="CustomerOrders[i].X"
CustomerOrders[i].NoOfProduct is null	Field not present in form	Add <input> field for NoOfProduct
CustomerName is null	Not posted from form	It shouldn't be — populate it server-side using a join
