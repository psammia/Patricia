public class Customer
{
    public int CustomerId { get; set; }
    public string Name { get; set; } = string.Empty;
}


public class Order
{
    public int OrderId { get; set; }
    public string Description { get; set; } = string.Empty;
    public decimal Cost { get; set; }
    public decimal Profit { get; set; }
    public List<CustomerOrder>? CustomerOrders { get; set; }
}


public class CustomerOrder
{
    public int CustomerId { get; set; }
    public int OrderId { get; set; }
    public bool IsPaid { get; set; }

    // For convenience
    public string? CustomerName { get; set; }
}


public class CustomerController : Controller
{
    private readonly ICustomerRepository _repo;

    public CustomerController(ICustomerRepository repo)
    {
        _repo = repo;
    }

    public async Task<IActionResult> Index()
    {
        var customers = await _repo.GetAllCustomersAsync();
        return View(customers);
    }

    public IActionResult Create() => View();

    [HttpPost]
    public async Task<IActionResult> Create(Customer customer)
    {
        if (ModelState.IsValid)
        {
            await _repo.AddCustomerAsync(customer);
            return RedirectToAction("Index");
        }
        return View(customer);
    }

    public async Task<IActionResult> Edit(int id)
    {
        var customer = await _repo.GetCustomerByIdAsync(id);
        return View(customer);
    }

    [HttpPost]
    public async Task<IActionResult> Edit(Customer customer)
    {
        await _repo.UpdateCustomerAsync(customer);
        return RedirectToAction("Index");
    }

    public async Task<IActionResult> Delete(int id)
    {
        await _repo.DeleteCustomerAsync(id);
        return RedirectToAction("Index");
    }
}

public class OrderController : Controller
{
    private readonly IOrderRepository _repo;
    private readonly ICustomerRepository _customerRepo;

    public OrderController(IOrderRepository repo, ICustomerRepository customerRepo)
    {
        _repo = repo;
        _customerRepo = customerRepo;
    }

    public async Task<IActionResult> Index()
    {
        var orders = await _repo.GetAllOrdersAsync();
        return View(orders);
    }

    public async Task<IActionResult> Create()
    {
        ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
        return View();
    }

    [HttpPost]
    public async Task<IActionResult> Create(Order order, int[] selectedCustomers)
    {
        if (ModelState.IsValid)
        {
            await _repo.AddOrderWithCustomersAsync(order, selectedCustomers);
            return RedirectToAction("Index");
        }

        ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
        return View(order);
    }

    public async Task<IActionResult> Edit(int id)
    {
        var order = await _repo.GetOrderByIdAsync(id);
        ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
        return View(order);
    }

    [HttpPost]
    public async Task<IActionResult> Edit(Order order, int[] selectedCustomers)
    {
        await _repo.UpdateOrderWithCustomersAsync(order, selectedCustomers);
        return RedirectToAction("Index");
    }

    public async Task<IActionResult> Delete(int id)
    {
        await _repo.DeleteOrderAsync(id);
        return RedirectToAction("Index");
    }
}


@model IEnumerable<Customer>
<h2>Customers</h2>
<a href="/Customer/Create">Add New</a>
<table>
    <thead><tr><th>Name</th><th>Actions</th></tr></thead>
    <tbody>
    @foreach (var c in Model)
    {
        <tr>
            <td>@c.Name</td>
            <td>
                <a href="/Customer/Edit/@c.CustomerId">Edit</a> |
                <a href="/Customer/Delete/@c.CustomerId">Delete</a>
            </td>
        </tr>
    }
    </tbody>
</table>




@model Customer
<form asp-action="@(Model.CustomerId == 0 ? "Create" : "Edit")">
    <label>Name</label>
    <input asp-for="Name" />
    <button type="submit">Save</button>
</form>


@model IEnumerable<Order>
<h2>Orders</h2>
<a href="/Order/Create">Add New</a>
<table>
    <thead><tr><th>Description</th><th>Cost</th><th>Profit</th><th>Actions</th></tr></thead>
    <tbody>
    @foreach (var o in Model)
    {
        <tr>
            <td>@o.Description</td>
            <td>@o.Cost</td>
            <td>@o.Profit</td>
            <td>
                <a href="/Order/Edit/@o.OrderId">Edit</a> |
                <a href="/Order/Delete/@o.OrderId">Delete</a>
            </td>
        </tr>
    }
    </tbody>
</table>


@model Order
@{
    var customers = ViewBag.Customers as List<Customer>;
}
<form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")">
    <label>Description</label>
    <input asp-for="Description" /><br/>
    <label>Cost</label>
    <input asp-for="Cost" /><br/>
    <label>Profit</label>
    <input asp-for="Profit" /><br/>

    <label>Select Customers</label><br/>
    @foreach (var customer in customers)
    {
        <input type="checkbox" name="selectedCustomers" value="@customer.CustomerId" /> @customer.Name <br/>
    }

    <button type="submit">Save</button>
</form>



