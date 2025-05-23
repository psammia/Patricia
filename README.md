OrderController.cs
using Microsoft.AspNetCore.Mvc;

using OrdersTracking.Models;
using OrdersTracking.Repositories;

namespace OrdersTracking.Controllers
{
    public class OrderController : Controller
    {
        private readonly IOrderRepository _repo;
        private readonly ICustomerRepository _customerRepo;
        private readonly IStatusRepository _statusRepo;

        public OrderController(IOrderRepository repo, ICustomerRepository customerRepo, IStatusRepository statusRepo)
        {
            _repo = repo;
            _customerRepo = customerRepo;
            _statusRepo = statusRepo;
        }

        public async Task<IActionResult> Index()
        {
            var orders = await _repo.GetAllOrdersAsync();
            return View(orders);
        }

        public async Task<IActionResult> Create()
        {

            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(new Order());
        }

        [HttpPost]
        public async Task<IActionResult> Create(Order order)
        {
            if (ModelState.IsValid)
            {
                await _repo.AddOrderWithCustomersAsync(order);
                return RedirectToAction("Index");
            }

            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync(); 
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(order);
        }

        public async Task<IActionResult> Edit(int id)
        {
            var order = await _repo.GetOrderByIdAsync(id);
            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(order);
        }

        [HttpPost]
        public async Task<IActionResult> Edit(Order order, int[] selectedCustomers)
        {
            await _repo.UpdateOrderWithCustomersAsync(order, selectedCustomers);
            return RedirectToAction("Index");
        }

        public async Task<IActionResult> DeleteConfirmed(int id)
        {
            await _repo.DeleteOrderAsync(id);
            return RedirectToAction("Index");
        }
    }
}

OrderRepository.cs
using System.Data;
using System.Data.SqlClient;
using System.Transactions;

using Dapper;

using OrdersTracking.Models;

namespace OrdersTracking.Repositories
{
    public class OrderRepository : IOrderRepository
    {
        private readonly IConfiguration _config;

        public OrderRepository(IConfiguration config)
        {
            _config = config;
        }

        private IDbConnection Connection => new SqlConnection(_config.GetConnectionString("DefaultConnection"));

        public async Task<IEnumerable<Order>> GetAllOrdersAsync()
        {
            using var conn = Connection;
            return await conn.QueryAsync<Order>("SELECT * FROM Orders");
        }

        public async Task<Order?> GetOrderByIdAsync(int id)
        {
            using var conn = Connection;

            var order = await conn.QuerySingleOrDefaultAsync<Order>("SELECT * FROM Orders WHERE OrderId = @id", new { id });
            if (order != null)
            {
                var customerOrders = await conn.QueryAsync<CustomerOrder>(
                    @"SELECT co.CustomerId, co.OrderId, co.IsPaid, c.Name AS CustomerName
                  FROM CustomerOrders co
                  INNER JOIN Customers c ON c.CustomerId = co.CustomerId
                  WHERE co.OrderId = @id", new { id });

                order.CustomerOrders = customerOrders.ToList();
            }

            return order;
        }

        public async Task AddOrderWithCustomersAsync(Order order)
        {
            using var conn = Connection;
            conn.Open();
            using var tran = conn.BeginTransaction();

            try
            {
                var orderId = await Connection.ExecuteScalarAsync<int>(
                    "INSERT INTO Orders (OrderDate, Cost, Profit, NoOfProduct, TotalAmount, StatusCode) OUTPUT INSERTED.OrderId VALUES (@OrderDate, @Cost, @Profit, @NoOfProduct, @TotalAmount, @StatusCode)",
                    order, tran);

                foreach (var co in order.CustomerOrders)
                {
                    await Connection.ExecuteAsync(
                        "INSERT INTO CustomerOrder (OrderId, CustomerId, Amount, IsPaid) VALUES (@OrderId, @CustomerId, @Amount, @IsPaid)",
                        new { OrderId = orderId, CustomerId = co.CustomerId, Amount = co.Amount, IsPaid = co.IsPaid }, tran);
                }

                tran.Commit();
            }
            catch
            {
                tran.Rollback();
                throw;
            }
        }

        public async Task UpdateOrderWithCustomersAsync(Order order, int[] customerIds)
        {
            using var conn = Connection;
            conn.Open();
            using var tran = conn.BeginTransaction();

            try
            {
                await conn.ExecuteAsync("UPDATE Orders SET OrderDate = @OrderDate, Cost = @Cost, Profit = @Profit, NoOfProduct=@NoOfProduct, TotalAmount=@TotalAmount, StatusCode=@StatusCode WHERE OrderId = @OrderId", order, tran);

                // Delete previous customer associations
                await conn.ExecuteAsync("DELETE FROM CustomerOrders WHERE OrderId = @OrderId", new { order.OrderId }, tran);

                // Reinsert
                foreach (var customerId in customerIds)
                {
                    await conn.ExecuteAsync("INSERT INTO CustomerOrders (OrderId, CustomerId, IsPaid) VALUES (@OrderId, @CustomerId, 0)",
                        new { OrderId = order.OrderId, CustomerId = customerId }, tran);
                }

                tran.Commit();
            }
            catch
            {
                tran.Rollback();
                throw;
            }
        }

        public async Task DeleteOrderAsync(int id)
        {
            using var conn = Connection;
            conn.Open();
            using var tran = conn.BeginTransaction();

            try
            {
                await conn.ExecuteAsync("DELETE FROM CustomerOrders WHERE OrderId = @id", new { id }, tran);
                await conn.ExecuteAsync("DELETE FROM Orders WHERE OrderId = @id", new { id }, tran);
                tran.Commit();
            }
            catch
            {
                tran.Rollback();
                throw;
            }
        }
    }
}

create.cshtml@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.AllCustomers as List<Customer> ?? new List<Customer>();
    var statuses = ViewBag.Statuses as List<Status> ?? new List<Status>();
}

<h2>Create Order</h2>

<div class="card">
    <div class="card-content collapse show">
        <div class="card-body">
            <form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")" method="post" class="needs-validation" novalidate>
                <div class="mb-3">
                    <input asp-for="OrderDate" type="date" class="form-control" value="@DateTime.Today.ToString("yyyy-MM-dd")" /><br />
                    <span asp-validation-for="OrderDate" class="text-danger"></span>
                </div>

                <div class="mb-3">
                    <label asp-for="Cost" class="form-label"></label>
                    <input asp-for="Cost" class="form-control" />
                    <span asp-validation-for="Cost" class="text-danger"></span>
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
                    <label asp-for="TotalAmount" class="form-label"></label>
                    <input asp-for="TotalAmount" class="form-control" />
                    <span asp-validation-for="TotalAmount" class="text-danger"></span>
                </div>

                <div class="mb-3">
                    <label asp-for="StatusCode">Status</label>
                    <select asp-for="StatusCode"
                            class="form-control"
                            asp-items="@(new SelectList(ViewBag.Statuses, "StatusCode", "StatusCode"))">
                        <option value="">-- Select Status --</option>
                    </select>
                    <span asp-validation-for="StatusCode" class="text-danger"></span>
                </div>

                <!-- Customer + Amount + Paid Repeater -->
                <div id="customerRepeater">
                    <div class="customer-entry mb-2">
                        <select class="form-select customer-select" name="CustomerOrders[0].CustomerId">
                            <option value="">-- Select Customer --</option>
                            @foreach (var c in customers)
                            {
                                <option value="@c.CustomerId">@c.Name</option>
                            }
                        </select>
                        <input name="CustomerOrders[0].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                        <input name="CustomerOrders[0].IsPaid" type="checkbox" class="form-check-input" />
                        <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                    </div>
                </div>
                <button type="button" class="btn btn-secondary" id="addCustomer">Add Another Customer</button>

                <button type="submit" class="btn btn-success">Save Order</button>
            </form>
        </div>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />

    <script>
        let index = 1;
        $('#addCustomer').on('click', function () {
            let newEntry = `
                <div class="customer-entry mb-2">
                    <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                        <option value="">-- Select Customer --</option>
                        ${@Html.Raw(Newtonsoft.Json.JsonConvert.SerializeObject(customers))
                            .Replace("\"", "\\\"")
                    .Replace("{", "{ ")
                    .Replace("}", " }")
                    .Replace(",", ", ")}
                    </select>
                    <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                    <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
                    <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                </div>`;
            $('#customerRepeater').append(newEntry);
            index++;
        });

        $(document).on('click', '.remove-entry', function () {
            $(this).closest('.customer-entry').remove();
        });

        $(document).ready(function () {
            $('.customer-select').select2();
        });
    </script>
}


