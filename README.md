🔹 Controller: CustomerOrdersController.cs
public class CustomerOrdersController : Controller
{
    private readonly ApplicationDbContext _context;

    public CustomerOrdersController(ApplicationDbContext context)
    {
        _context = context;
    }

    // GET: CustomerOrders/Create
    public IActionResult Create()
    {
        ViewBag.Customers = new SelectList(_context.Customers, "CustomerId", "Name");
        ViewBag.Orders = new SelectList(_context.Orders, "OrderId", "OrderNumber");
        return View();
    }

    // POST: CustomerOrders/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(CustomerOrder customerOrder)
    {
        if (ModelState.IsValid)
        {
            _context.Add(customerOrder);
            await _context.SaveChangesAsync();
            return RedirectToAction(nameof(Index)); // or wherever you want to go
        }
        ViewBag.Customers = new SelectList(_context.Customers, "CustomerId", "Name", customerOrder.CustomerId);
        ViewBag.Orders = new SelectList(_context.Orders, "OrderId", "OrderNumber", customerOrder.OrderId);
        return View(customerOrder);
    }

    // Optionally: Index, Details, Edit, Delete
}



🔹 View: Create.cshtml
@model YourNamespace.Models.CustomerOrder

@{
    ViewData["Title"] = "Add Customer to Order";
}

<h2>Add Customer to Order</h2>

<form asp-action="Create" method="post">
    <div class="form-group">
        <label asp-for="CustomerId"></label>
        <select asp-for="CustomerId" class="form-control" asp-items="ViewBag.Customers"></select>
    </div>
    <div class="form-group">
        <label asp-for="OrderId"></label>
        <select asp-for="OrderId" class="form-control" asp-items="ViewBag.Orders"></select>
    </div>
    <div class="form-group">
        <label asp-for="Amount"></label>
        <input asp-for="Amount" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="NoOfProducts"></label>
        <input asp-for="NoOfProducts" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="IsPaid"></label>
        <input asp-for="IsPaid" type="checkbox" />
    </div>
    <button type="submit" class="btn btn-primary">Add</button>
</form>



🔹 Controller Additions (CustomerOrdersController.cs)
// GET: CustomerOrders
public async Task<IActionResult> Index()
{
    var customerOrders = _context.CustomerOrders
        .Include(co => co.Customer)
        .Include(co => co.Order);

    return View(await customerOrders.ToListAsync());
}

// GET: CustomerOrders/Edit/5
public async Task<IActionResult> Edit(int? id)
{
    if (id == null)
        return NotFound();

    var customerOrder = await _context.CustomerOrders.FindAsync(id);
    if (customerOrder == null)
        return NotFound();

    ViewBag.Customers = new SelectList(_context.Customers, "CustomerId", "Name", customerOrder.CustomerId);
    ViewBag.Orders = new SelectList(_context.Orders, "OrderId", "OrderNumber", customerOrder.OrderId);

    return View(customerOrder);
}

// POST: CustomerOrders/Edit/5
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(int id, CustomerOrder customerOrder)
{
    if (id != customerOrder.CustomerOrderId)
        return NotFound();

    if (ModelState.IsValid)
    {
        try
        {
            _context.Update(customerOrder);
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!_context.CustomerOrders.Any(e => e.CustomerOrderId == id))
                return NotFound();
            else
                throw;
        }
        return RedirectToAction(nameof(Index));
    }

    ViewBag.Customers = new SelectList(_context.Customers, "CustomerId", "Name", customerOrder.CustomerId);
    ViewBag.Orders = new SelectList(_context.Orders, "OrderId", "OrderNumber", customerOrder.OrderId);

    return View(customerOrder);
}

🔹 View: Index.cshtml (List)
@model IEnumerable<YourNamespace.Models.CustomerOrder>

@{
    ViewData["Title"] = "Customer Orders";
}

<h2>Customer Orders</h2>

<table class="table table-bordered">
    <thead>
        <tr>
            <th>Customer</th>
            <th>Order</th>
            <th>Amount</th>
            <th>No. of Products</th>
            <th>Is Paid</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
    @foreach (var item in Model)
    {
        <tr>
            <td>@item.Customer?.Name</td>
            <td>@item.Order?.OrderNumber</td>
            <td>@item.Amount</td>
            <td>@item.NoOfProducts</td>
            <td>@item.IsPaid</td>
            <td>
                <a asp-action="Edit" asp-route-id="@item.CustomerOrderId" class="btn btn-sm btn-warning">Edit</a>
            </td>
        </tr>
    }
    </tbody>
</table>


 View: Edit.cshtml
@model YourNamespace.Models.CustomerOrder

@{
    ViewData["Title"] = "Edit Customer Order";
}

<h2>Edit Customer Order</h2>

<form asp-action="Edit">
    <input type="hidden" asp-for="CustomerOrderId" />

    <div class="form-group">
        <label asp-for="CustomerId"></label>
        <select asp-for="CustomerId" class="form-control" asp-items="ViewBag.Customers"></select>
    </div>
    <div class="form-group">
        <label asp-for="OrderId"></label>
        <select asp-for="OrderId" class="form-control" asp-items="ViewBag.Orders"></select>
    </div>
    <div class="form-group">
        <label asp-for="Amount"></label>
        <input asp-for="Amount" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="NoOfProducts"></label>
        <input asp-for="NoOfProducts" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="IsPaid"></label>
        <input asp-for="IsPaid" type="checkbox" />
    </div>
    <button type="submit" class="btn btn-success">Save</button>
    <a asp-action="Index" class="btn btn-secondary">Cancel</a>
</form>



🔹 Add to Controller (CustomerOrdersController.cs)
// GET: CustomerOrders/Delete/5
public async Task<IActionResult> Delete(int? id)
{
    if (id == null)
        return NotFound();

    var customerOrder = await _context.CustomerOrders
        .Include(c => c.Customer)
        .Include(o => o.Order)
        .FirstOrDefaultAsync(m => m.CustomerOrderId == id);

    if (customerOrder == null)
        return NotFound();

    return View(customerOrder);
}

// POST: CustomerOrders/Delete/5
[HttpPost, ActionName("Delete")]
[ValidateAntiForgeryToken]
public async Task<IActionResult> DeleteConfirmed(int id)
{
    var customerOrder = await _context.CustomerOrders.FindAsync(id);
    if (customerOrder != null)
    {
        _context.CustomerOrders.Remove(customerOrder);
        await _context.SaveChangesAsync();
    }

    return RedirectToAction(nameof(Index));
}


🔹 View: Delete.cshtml
@model YourNamespace.Models.CustomerOrder

@{
    ViewData["Title"] = "Delete Customer Order";
}

<h2>Delete Customer Order</h2>

<div>
    <h4>Are you sure you want to delete this customer order?</h4>
    <hr />
    <dl class="row">
        <dt class="col-sm-2">Customer</dt>
        <dd class="col-sm-10">@Model.Customer?.Name</dd>

        <dt class="col-sm-2">Order</dt>
        <dd class="col-sm-10">@Model.Order?.OrderNumber</dd>

        <dt class="col-sm-2">Amount</dt>
        <dd class="col-sm-10">@Model.Amount</dd>

        <dt class="col-sm-2">No. of Products</dt>
        <dd class="col-sm-10">@Model.NoOfProducts</dd>

        <dt class="col-sm-2">Is Paid</dt>
        <dd class="col-sm-10">@Model.IsPaid</dd>
    </dl>

    <form asp-action="Delete">
        <input type="hidden" asp-for="CustomerOrderId" />
        <button type="submit" class="btn btn-danger">Delete</button>
        <a asp-action="Index" class="btn btn-secondary">Cancel</a>
    </form>
</div>



Modify the last column in your Index.cshtml:
<td>
    <a asp-action="Edit" asp-route-id="@item.CustomerOrderId" class="btn btn-sm btn-warning">Edit</a>
    <a asp-action="Delete" asp-route-id="@item.CustomerOrderId" class="btn btn-sm btn-danger">Delete</a>
</td>



