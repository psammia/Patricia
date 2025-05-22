
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
}


Create.Cshtml
@model Order
@{
    var customers = ViewBag.Customers as List<Customer>;
}
<form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")">

    <label>Order Date</label>
    <input asp-for="OrderDate" /><br />

    <label>Cost</label>
    <input asp-for="Cost" /><br />

    <label>Profit</label>
    <input asp-for="Profit" /><br />

    <label>NoOfProduct</label>
    <input asp-for="NoOfProduct" /><br />

    <label>Total Amount</label>
    <input asp-for="TotalAmount" /><br />

    <label>Status</label>
    <input asp-for="StatusCode" /><br />

    <label>Select Customers</label><br />
    @foreach (var customer in customers)
    {
        <input type="checkbox" name="selectedCustomers" value="@customer.CustomerId" /> @customer.Name <br />
    }

    <button type="submit">Save</button>
</form>

Order.cs model
namespace OrdersTracking.Models
{
    public class Order
    {
        public int OrderId { get; set; }
        public DateTime OrderDate { get; set; }
        public decimal Cost { get; set; }
        public decimal Profit { get; set; }
        public Int32? NoOfProduct { get; set; }
        public decimal? TotalAmount { get; set; }
        public List<CustomerOrder>? CustomerOrders { get; set; }
        public string StatusCode { get; set; } = string.Empty;
    }
}

