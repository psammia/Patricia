using Microsoft.AspNetCore.Mvc;
using System.Data;
using System.Data.SqlClient;
using OrderTracking.Models;
using System;

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
::contentReference[oaicite:0]{ index = 0}
        }
    }
}
