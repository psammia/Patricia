Order.cs
using System.ComponentModel.DataAnnotations;
using System.Globalization;

using Microsoft.VisualBasic;

namespace OrdersTracking.Models
{
    public class Order
    {
        public int OrderId { get; set; }
        [Required]
        [DataType(DataType.Date)]
        [DisplayFormat(DataFormatString = "{0:yyyy-MM-dd}", ApplyFormatInEditMode = true)]
        public DateTime OrderDate { get; set; }
        //[Required]
        //public decimal? Cost { get; set; }
        [Required]
        public decimal? Profit { get; set; }
        [Required]
        public Int32? NoOfProduct { get; set; }
        [Required]
        public decimal? TotalAmount { get; set; }
      
        [Required]
        public string? StatusCode { get; set; }

        public List<CustomerOrder> CustomerOrders { get; set; } = new();
    }
}


CustomerOrder.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace OrdersTracking.Models
{
    public class CustomerOrder
    {
        public int CustomerId { get; set; }
        public int OrderId { get; set; }
        
        [Required]
        public bool IsPaid { get; set; }
        [Required]
        public decimal? Amount { get; set; }

        [Required]
        public int? NoOfProductperCustomer { get; set; }

        //// For convenience
        [NotMapped]
        public string? customername { get; set; }
    }
}

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

            var customers = await _customerRepo.GetAllCustomersAsync();
            var statuses = await _statusRepo.GetAllStatusesAsync();
            ViewBag.Customers = customers ?? new List<Customer>();
            ViewBag.Statuses = statuses ?? new List<Status>();
            return View(new Order());
        }

        [HttpPost]
        public async Task<IActionResult> Create(Order order)
        {
            
            if(order.CustomerOrders == null || !order.CustomerOrders.Any())
            {
                ModelState.AddModelError("", "Please add at least one customer.");
            }
            if (!ModelState.IsValid)
            {
                ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
                ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
                return View(order);

            }
            await _repo.AddOrderWithCustomersAsync(order);
            return RedirectToAction("Index");
        }

        public async Task<IActionResult> Edit(int id)
        {
            var order = await _repo.GetOrderByIdAsync(id);
            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(order);
        }

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
                var orderId = await conn.ExecuteScalarAsync<int>(
                    "INSERT INTO Orders (OrderDate, Profit, NoOfProduct, TotalAmount, StatusCode) OUTPUT INSERTED.OrderId VALUES (@OrderDate, @Profit, @NoOfProduct, @TotalAmount, @StatusCode)",
                    order, tran);

                foreach (CustomerOrder co in order.CustomerOrders)
                {
                    await conn.ExecuteAsync(
                        "INSERT INTO CustomerOrders (OrderId, CustomerId, Amount, IsPaid, NoOfProductperCustomer) VALUES (@OrderId, @CustomerId, @Amount, @IsPaid,@NoOfProductperCustomer)",
                        new { OrderId = orderId, CustomerId = co.CustomerId, Amount = co.Amount, IsPaid = co.IsPaid, NoOfProductperCustomer = co.NoOfProductperCustomer }, tran);
                }

                tran.Commit();
            }
            catch
            {
                tran.Rollback();
                throw;
            }
        }

        public async Task UpdateOrderWithCustomersAsync(Order order)
        {
            using var conn = Connection;
            conn.Open();
            using var tran = conn.BeginTransaction();

            try
            {
                await conn.ExecuteAsync(
                    "UPDATE Orders SET OrderDate = @OrderDate, Profit = @Profit, NoOfProduct = @NoOfProduct, TotalAmount = @TotalAmount, StatusCode = @StatusCode WHERE OrderId = @OrderId",
                    order, tran);

                await conn.ExecuteAsync("DELETE FROM CustomerOrders WHERE OrderId = @OrderId", new { order.OrderId }, tran);

                foreach (var co in order.CustomerOrders)
                {
                    await conn.ExecuteAsync(
                        "INSERT INTO CustomerOrders (OrderId, CustomerId, Amount, IsPaid, NoOfProductperCustomer) VALUES (@OrderId, @CustomerId, @Amount, @IsPaid, @NoOfProductperCustomer)",
                        new { OrderId = order.OrderId, co.CustomerId, co.Amount, co.IsPaid, co.NoOfProductperCustomer }, tran);
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

Edit.cshtml
@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Edit Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
    var customerOptionsHtml = new System.Text.StringBuilder();
    foreach (var customer in customers)
    {
        customerOptionsHtml.Append($"<option value=\"{customer.CustomerId}\">{customer.Name}</option>");
    }
}

<h2>Edit Order</h2>

<form asp-action="Edit" method="post">
    <input type="hidden" asp-for="OrderId" />

    <div class="mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" type="date" />
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
            <option value="">-- Select Status --</option>
            @foreach (var status in (IEnumerable<Status>)ViewBag.Statuses)
            {
                <option value="@status.StatusCode">@status.StatusCode</option>
            }
        </select>
        <span asp-validation-for="StatusCode" class="text-danger"></span>
    </div>

    <h4>Customers</h4>
    <div id="customerRepeater" class="mb-3">
        @for (int i = 0; i < Model.CustomerOrders.Count; i++)
        {
            var co = Model.CustomerOrders[i];
            <div class="customer-entry mb-3 border p-3 rounded">
                <label>Customer</label>
                <select class="form-select" name="CustomerOrders[@i].CustomerId" required>
                    <option value="">-- Select Customer --</option>
                    @foreach (var customer in customers)
                    {
                        <option value="@customer.CustomerId">@customer.Name</option>
                    }
                </select>

                <label class="mt-2">Amount</label>
                <input name="CustomerOrders[@i].Amount" class="form-control" value="@co.Amount" required />

                <label class="mt-2">No. of Product</label>
                <input name="CustomerOrders[@i].NoOfProductperCustomer" type="number" class="form-control" value="@co.NoOfProductperCustomer" required />

                <div class="form-check mt-2">
                    <input type="hidden" name="CustomerOrders[@i].IsPaid" value="false" />
                    <input name="CustomerOrders[@i].IsPaid" type="checkbox" value="true" class="form-check-input" @(co.IsPaid ? "checked" : "") />
                    <label class="form-check-label">Is Paid</label>
                </div>

                <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
            </div>
        }
    </div>

    <button type="button" id="addCustomer" class="btn btn-success mb-3">Add Customer</button>

    <div class="form-group">
        <input type="submit" value="Save" class="btn btn-primary" />
    </div>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = @Model.CustomerOrders.Count;
        const customerOptionsHtml = `@Html.Raw(customerOptionsHtml.ToString())`;

        function getCustomerEntryHtml(i) {
            return `
                <div class="customer-entry mb-3 border p-3 rounded">
                    <label>Customer</label>
                    <select class="form-select" name="CustomerOrders[${i}].CustomerId" required>
                        <option value="">-- Select Customer --</option>
                        ${customerOptionsHtml}
                    </select>

                    <label class="mt-2">Amount</label>
                    <input name="CustomerOrders[${i}].Amount" class="form-control" required />

                    <label class="mt-2">No. of Product</label>
                                    <input name="CustomerOrders[${i}].NoOfProductperCustomer" type="number" class="form-control" required />

                    <div class="form-check mt-2">
                        <input type="hidden" name="CustomerOrders[${i}].IsPaid" value="false" />
                        <input name="CustomerOrders[${i}].IsPaid" type="checkbox" value="true" class="form-check-input" />
                        <label class="form-check-label">Is Paid</label>
                    </div>

                    <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
                </div>
            `;
        }

        document.getElementById("addCustomer").addEventListener("click", function () {
            const container = document.getElementById("customerRepeater");
            container.insertAdjacentHTML("beforeend", getCustomerEntryHtml(index));
            index++;
        });

        document.getElementById("customerRepeater").addEventListener("click", function (e) {
            if (e.target.classList.contains("remove-entry")) {
                e.target.closest(".customer-entry").remove();
            }
        });
    </script>
}



