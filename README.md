{
  "ConnectionStrings": {
    "DefaultConnection": "Server=YOUR_SERVER_NAME;Database=OrdersTracking;Trusted_Connection=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}


public class Customer
{
    public int CustomerId { get; set; }
    public string Name { get; set; } = "";
    public string? Email { get; set; }
    public string? Phone { get; set; }
}

public class Order
{
    public int OrderId { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal Cost { get; set; }

    using Microsoft.AspNetCore.Mvc;
using System.Data;
using System.Data.SqlClient;
using OrdersTracking.Models;

public class CustomerController : Controller
{
    private readonly IConfiguration _configuration;
    private readonly string _connectionString;

    public CustomerController(IConfiguration configuration)
    {
        _configuration = configuration;
        _connectionString = _configuration.GetConnectionString("DefaultConnection");
    }

    public IActionResult Index()
    {
        List<Customer> customers = new List<Customer>();

        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new SqlCommand("SELECT * FROM Customers", conn);
            conn.Open();
            SqlDataReader reader = cmd.ExecuteReader();
            while (reader.Read())
            {
                customers.Add(new Customer
                {
                    CustomerId = (int)reader["CustomerId"],
                    Name = reader["Name"].ToString() ?? "",
                    Email = reader["Email"].ToString(),
                    Phone = reader["Phone"].ToString()
                });
            }
        }

        return View(customers);
    }

    public IActionResult Create()
    {
        return View();
    }

    [HttpPost]
    public IActionResult Create(Customer customer)
    {
        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new SqlCommand("UpsertCustomer", conn);
            cmd.CommandType = CommandType.StoredProcedure;

            SqlParameter idParam = new SqlParameter("@CustomerId", SqlDbType.Int)
            {
                Direction = ParameterDirection.Output
            };
            cmd.Parameters.Add(idParam);
            cmd.Parameters.AddWithValue("@Name", customer.Name);
            cmd.Parameters.AddWithValue("@Email", customer.Email ?? (object)DBNull.Value);
            cmd.Parameters.AddWithValue("@Phone", customer.Phone ?? (object)DBNull.Value);

            conn.Open();
            cmd.ExecuteNonQuery();
        }

        return RedirectToAction("Index");
    }

    public IActionResult Edit(int id)
    {
        Customer customer = new Customer();

        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new SqlCommand("SELECT * FROM Customers WHERE CustomerId = @CustomerId", conn);
            cmd.Parameters.AddWithValue("@CustomerId", id);
            conn.Open();
            SqlDataReader reader = cmd.ExecuteReader();
            if (reader.Read())
            {
                customer.CustomerId = (int)reader["CustomerId"];
                customer.Name = reader["Name"].ToString() ?? "";
                customer.Email = reader["Email"].ToString();
                customer.Phone = reader["Phone"].ToString();
            }
        }

        return View(customer);
    }

    [HttpPost]
    public IActionResult Edit(Customer customer)
    {
        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new SqlCommand("UpsertCustomer", conn);
            cmd.CommandType = CommandType.StoredProcedure;

            cmd.Parameters.AddWithValue("@CustomerId", customer.CustomerId);
            cmd.Parameters.AddWithValue("@Name", customer.Name);
            cmd.Parameters.AddWithValue("@Email", customer.Email ?? (object)DBNull.Value);
            cmd.Parameters.AddWithValue("@Phone", customer.Phone ?? (object)DBNull.Value);

            conn.Open();
            cmd.ExecuteNonQuery();
        }

        return RedirectToAction("Index");
    }
}

using Microsoft.AspNetCore.Mvc;
using System.Data;
using System.Data.SqlClient;
using OrdersTracking.Models;

public class OrderController : Controller
{
    private readonly IConfiguration _configuration;
    private readonly string _connectionString;

    public OrderController(IConfiguration configuration)
    {
        _configuration = configuration;
        _connectionString = _configuration.GetConnectionString("DefaultConnection");
    }

    public IActionResult Index()
    {
        List<Order> orders = new List<Order>();

        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new SqlCommand("SELECT * FROM Orders", conn);
            conn.Open();
            SqlDataReader reader = cmd.ExecuteReader();
            while (reader.Read())
            {
                orders.Add(new Order
                {
                    OrderId = (int)reader["OrderId"],
                    OrderDate = (DateTime)reader["OrderDate"],
                    Cost = (decimal)reader["Cost"],
                    Profit = (decimal)reader["Profit"],
                    IsPaid = (bool)reader["IsPaid"]
                });
            }
        }

        return View(orders);
    }

    public IActionResult Create()
    {
        return View();
    }

    [HttpPost]
    public IActionResult Create(Order order)
    {
        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new SqlCommand("UpsertOrder", conn);
            cmd.CommandType = CommandType.StoredProcedure;

            SqlParameter idParam = new SqlParameter("@OrderId", SqlDbType.Int)
            {
                Direction = ParameterDirection.Output
            };
            cmd.Parameters.Add(idParam);
            cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
            cmd.Parameters.AddWithValue("@Cost", order.Cost);
            cmd.Parameters.AddWithValue("@Profit", order.Profit);
            cmd.Parameters.AddWithValue("@IsPaid", order.IsPaid);

            conn.Open();
            cmd.ExecuteNonQuery();
        }

        return RedirectToAction("Index");
    }

    public IActionResult Edit(int id)
    {
        Order order = new Order();

        using (SqlConnection conn = new SqlConnection(_connectionString))
        {
            SqlCommand cmd = new Sql
::contentReference[oaicite:0]{index=0}
 


    public decimal Profit { get; set; }
    public bool IsPaid { get; set; }
}
