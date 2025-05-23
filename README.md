✅ How to Fix
1. Ensure ViewBag.AllCustomers is populated in your controller before returning the view
In your OrderController, for the Create GET action, you must assign the ViewBag.AllCustomers like this:

csharp
Copy
Edit
public IActionResult Create()
{
    var customers = _yourDbContext.Customers.ToList(); // or however you're getting customers
    var statuses = _yourDbContext.Statuses.ToList(); // if needed

    ViewBag.AllCustomers = customers;
    ViewBag.Statuses = statuses;

    return View(new Order()); // or return View(orderViewModel);
}
2. Add a null check fallback in the view (optional but safe)
If you want to prevent the Razor view from breaking even if ViewBag.AllCustomers is null:

csharp
Copy
Edit
@{
    var customers = ViewBag.AllCustomers as List<Customer> ?? new List<Customer>();
    var statuses = ViewBag.Statuses as List<Status> ?? new List<Status>();
}
⚠️ Your JavaScript block has a problem
In this part:

js
Copy
Edit
${@Html.Raw(Newtonsoft.Json.JsonConvert.SerializeObject(customers))
    .Replace("\"", "\\\"")
    .Replace("{", "{ ")
    .Replace("}", " }")
    .Replace(",", ", ")}
This won't work as expected in a Razor file. Razor doesn't support inline C# expressions in JavaScript template literals this way.

✅ Safer Alternative for Customer List in JS
Instead, serialize the customers into a JS variable above the script:

html
Copy
Edit
<script>
    const customersList = @Html.Raw(Json.Serialize(ViewBag.AllCustomers));
</script>
Then use that customersList in your JS code to dynamically build <option> tags.

js
Copy
Edit
let index = 1;

$('#addCustomer').on('click', function () {
    let customerOptions = '<option value="">-- Select Customer --</option>';
    customersList.forEach(c => {
        customerOptions += `<option value="${c.customerId}">${c.name}</option>`;
    });

    let newEntry = `
        <div class="customer-entry mb-2">
            <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                ${customerOptions}
            </select>
            <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
            <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
            <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
        </div>`;

    $('#customerRepeater').append(newEntry);
    index++;
});
✅ Summary of Fixes
In your controller:

csharp
Copy
Edit
ViewBag.AllCustomers = _context.Customers.ToList();
ViewBag.Statuses = _context.Statuses.ToList();
In your view:

csharp
Copy
Edit
var customers = ViewBag.AllCustomers as List<Customer> ?? new List<Customer>();
Move JSON serialization to a JS variable:

html
Copy
Edit
<script>
    const customersList = @Html.Raw(Json.Serialize(ViewBag.AllCustomers));
</script>
Let me know if you'd like me to correct t
