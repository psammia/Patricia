✅ 2. Create an IStatusRepository Interface
using OrdersTracking.Models;

public interface IStatusRepository
{
    Task<IEnumerable<Status>> GetAllStatusesAsync();
}


✅ 3. Implement StatusRepository using Dapper
using Dapper;
using OrdersTracking.Models;
using System.Data;
using System.Data.SqlClient;

public class StatusRepository : IStatusRepository
{
    private readonly IDbConnection _db;

    public StatusRepository(IConfiguration config)
    {
        _db = new SqlConnection(config.GetConnectionString("DefaultConnection"));
    }

    public async Task<IEnumerable<Status>> GetAllStatusesAsync()
    {
        string sql = "SELECT StatusCode FROM Statuses";
        return await _db.QueryAsync<Status>(sql);
    }
}



 4. Inject IStatusRepository in OrderController
Update your constructor:
private readonly IStatusRepository _statusRepo;

public OrderController(IOrderRepository repo, ICustomerRepository customerRepo, IStatusRepository statusRepo)
{
    _repo = repo;
    _customerRepo = customerRepo;
    _statusRepo = statusRepo;
}


public async Task<IActionResult> Create()
{
    ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
    ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
    return View();
}

public async Task<IActionResult> Edit(int id)
{
    var order = await _repo.GetOrderByIdAsync(id);
    ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
    ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
    return View(order);
}




Update Create() and Edit() GET methods to pass statuses to the view:
ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();


 5.5. Update the View (Create.cshtml / Edit.cshtml)
Replace the input for StatusCode with:

<label>Status</label>
<select asp-for="StatusCode" class="form-control">
    <option value="">-- Select Status --</option>
    @foreach (var status in (IEnumerable<Status>)ViewBag.Statuses)
    {
        <option value="@status.StatusCode">@status.StatusCode</option>
    }
</select>
<span asp-validation-for="StatusCode" class="text-danger"></span>




 6. Register the StatusRepository in Program.cs
builder.Services.AddScoped<IStatusRepository, StatusRepository>();




