create.cshtml
@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
    var statuses = ViewBag.Statuses as List<Status> ?? new List<Status>();
}

<h2>Create Order</h2>

<div class="card">
    <div class="card-content collapse show">
        <div class="card-body">
            <form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")" method="post" class="needs-validation" novalidate>
                @* Date *@
                <div class="mb-3">
                    <label asp-for="OrderDate" class="form-label"></label>
                    <input asp-for="OrderDate" type="date" class="form-control" value="@DateTime.Today.ToString("yyyy-MM-dd")" />
                    <span asp-validation-for="OrderDate" class="text-danger"></span>
                </div>

                @* TotalAmount *@
                <div class="mb-3">
                    <label asp-for="TotalAmount" class="form-label"></label>
                    <input asp-for="TotalAmount" class="form-control" />
                    <span asp-validation-for="TotalAmount" class="text-danger"></span>
                </div>

                @* NoOfProduct *@
                <div class="mb-3">
                    <label asp-for="NoOfProduct" class="form-label"></label>
                    <input asp-for="NoOfProduct" class="form-control" />
                    <span asp-validation-for="NoOfProduct" class="text-danger"></span>
                </div>

                @* Profit *@
                <div class="mb-3">
                    <label asp-for="Profit" class="form-label"></label>
                    <input asp-for="Profit" class="form-control" />
                    <span asp-validation-for="Profit" class="text-danger"></span>
                </div>

                @* Status dropdown *@
                <div class="mb-3">
                    <label asp-for="StatusCode" class="form-label">Status</label>
                    <select asp-for="StatusCode" class="form-control" asp-items="@(new SelectList(statuses, "StatusCode", "StatusCode"))">
                        <option value="">-- Select Status --</option>
                    </select>
                    <span asp-validation-for="StatusCode" class="text-danger"></span>
                </div>

                @* Customer/Amount/IsPaid Repeater *@
                <div id="customerRepeater">
                    <div class="customer-entry mb-2">
                        <select class="form-select customer-select" name="CustomerOrders[0].CustomerId">
                            <option value="">-- Select Customer --</option>
                            @foreach (var c in customers)
                            {
                                <option value="@c.CustomerId">@c.Name</option>
                            }
                        </select>

                        <input name="CustomerOrders[0].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1 mt-3" style="width: 600px;" />

                        <label class="form-check-label mt-3">
                            <input name="CustomerOrders[0].IsPaid" type="checkbox" class="form-check-input" />
                            Is Paid
                        </label>

                        <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                    </div>
                </div>

                <button type="button" class="btn btn-secondary mt-2" id="addCustomer">+ Add Customer</button>

                <div class="mt-4">
                    <button type="submit" class="btn btn-success">Save Order</button>
                </div>
            </form>
        </div>
    </div>
</div>

@section Scripts {
    <script>
        let index = 1;

        const customerOptionsHtml = `@Html.Raw(string.Join("", customers.Select(c => $"<option value='{c.CustomerId}'>{c.Name}</option>")))`;

        $('#addCustomer').on('click', function () {
            const entryHtml = `
                        <div class="customer-entry mb-2">
                            <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                                <option value="">-- Select Customer --</option>
                                ${customerOptionsHtml}
                            </select>

                            <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1 mt-3" style="width: 600px;" />

                            <label class="form-check-label mt-3">
                                <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
                                Is Paid
                            </label>

                            <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                        </div>
                    `;
            $('#customerRepeater').append(entryHtml);
            index++;
        });

        $(document).on('click', '.remove-entry', function () {
            $(this).closest('.customer-entry').remove();
        });

        $(document).ready(function () {
            $('.customer-select').select2(); // if using select2
        });
    </script>
}

order controller
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



