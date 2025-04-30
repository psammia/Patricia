// Project Structure Hierarchy

// Models Folder

Models/

Customer.cs

Order.cs

OrderItem.cs

Payment.cs



// Data Access Helper

Helpers/

DatabaseHelper.cs



// Repositories Folder

Repositories/

CustomerRepository.cs

OrderRepository.cs

OrderItemRepository.cs

PaymentRepository.cs



// Controllers Folder

Controllers/

CustomerController.cs

OrderController.cs

OrderItemController.cs

PaymentController.cs



// Views Folder

Views/

Customer/

Index.cshtml

Create.cshtml


Order/

Index.cshtml

Create.cshtml


OrderItem/

Index.cshtml

Create.cshtml


Payment/

Index.cshtml

Create.cshtml




// Configuration Files

appsettings.json


// Entry Point

Program.cs


// MVC Model Definitions

// Customer.cs public class Customer { public int CustomerId { get; set; } public string Name { get; set; } public string Email { get; set; } public string Phone { get; set; } }

// Order.cs public class Order { public int OrderId { get; set; } public int CustomerId { get; set; } public DateTime OrderDate { get; set; } public decimal TotalCost { get; set; } public decimal Profit { get; set; } public bool IsPaid { get; set; } }

// OrderItem.cs public class OrderItem { public int OrderItemId { get; set; } public int OrderId { get; set; } public string ItemName { get; set; } public int Quantity { get; set; } public decimal UnitPrice { get; set; } public decimal UnitCost { get; set; } }

// Payment.cs public class Payment { public int PaymentId { get; set; } public int OrderId { get; set; } public DateTime PaymentDate { get; set; } public decimal AmountPaid { get; set; } }

// OrderRepository.cs public class OrderRepository { private readonly string _connectionString;

public OrderRepository(IConfiguration configuration)
{
    _connectionString = configuration.GetConnectionString("DefaultConnection");
}

public void AddOrder(Order order)
{
    using (SqlConnection conn = new SqlConnection(_connectionString))
    using (SqlCommand cmd = new SqlCommand("sp_AddOrder", conn))
    {
        cmd.CommandType = CommandType.StoredProcedure;
        cmd.Parameters.AddWithValue("@CustomerId", order.CustomerId);
        cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
        cmd.Parameters.AddWithValue("@TotalCost", order.TotalCost);
        cmd.Parameters.AddWithValue("@Profit", order.Profit);
        cmd.Parameters.AddWithValue("@IsPaid", order.IsPaid);

        conn.Open();
        cmd.ExecuteNonQuery();
    }
}

public List<Order> GetAllOrders()
{
    List<Order> orders = new List<Order>();

    using (SqlConnection conn = new SqlConnection(_connectionString))
    using (SqlCommand cmd = new SqlCommand("sp_GetAllOrders", conn))
    {
        cmd.CommandType = CommandType.StoredProcedure;
        conn.Open();

        using (SqlDataReader reader = cmd.ExecuteReader())
        {
            while (reader.Read())
            {
                orders.Add(new Order
                {
                    OrderId = Convert.ToInt32(reader["OrderId"]),
                    CustomerId = Convert.ToInt32(reader["CustomerId"]),
                    OrderDate = Convert.ToDateTime(reader["OrderDate"]),
                    TotalCost = Convert.ToDecimal(reader["TotalCost"]),
                    Profit = Convert.ToDecimal(reader["Profit"]),
                    IsPaid = Convert.ToBoolean(reader["IsPaid"])
                });
            }
        }
    }

    return orders;
}

}

// OrderController.cs public class OrderController : Controller { private readonly OrderRepository _orderRepo;

public OrderController(IConfiguration config)
{
    _orderRepo = new OrderRepository(config);
}

public IActionResult Index()
{
    var orders = _orderRepo.GetAllOrders();
    return View(orders);
}

public IActionResult Create()
{
    return View();
}

[HttpPost]
public IActionResult Create(Order order)
{
    if (ModelState.IsValid)
    {
        _orderRepo.AddOrder(order);
        return RedirectToAction("Index");
    }
    return View(order);
}

}

// Views/Order/Index.cshtml @model IEnumerable<Order>

<h2>Orders</h2>
<table>
    <tr>
        <th>ID</th>
        <th>Date</th>
        <th>Total</th>
        <th>Profit</th>
        <th>Paid</th>
    </tr>
    @foreach (var item in Model)
    {
        <tr>
            <td>@item.OrderId</td>
            <td>@item.OrderDate</td>
            <td>@item.TotalCost</td>
            <td>@item.Profit</td>
            <td>@item.IsPaid</td>
        </tr>
    }
</table>// Views/Order/Create.cshtml @model Order

<h2>Create Order</h2>
<form asp-action="Create">
    <div>
        <label>Customer ID</label>
        <input asp-for="CustomerId" />
    </div>
    <div>
        <label>Order Date</label>
        <input asp-for="OrderDate" type="date" />
    </div>
    <div>
        <label>Total Cost</label>
        <input asp-for="TotalCost" />
    </div>
    <div>
        <label>Profit</label>
        <input asp-for="Profit" />
    </div>
    <div>
        <label>Is Paid</label>
        <input asp-for="IsPaid" type="checkbox" />
    </div>
    <button type="submit">Save</button>
</form>// Next step: OrderItem and Payment code and stored procedures

