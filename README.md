using System.Data;
using Microsoft.AspNetCore.Mvc;
using OrderTracking.Models;
using System.Data.SqlClient;

namespace OrdersTracking.Controllers
{
    public class OrderController : Controller
    {
        private readonly IConfiguration _config;
        private readonly string _connectionString;

        public OrderController(IConfiguration config)
        {
            _config = config;
            _connectionString = _config.GetConnectionString("DefaultConnection")!;
        }

        [Obsolete]
        public IActionResult Index()
        {
            var orders = new List<Order>();
            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                using (SqlCommand cmd = new SqlCommand("SELECT * FROM Orders", con))
                {
                    using (SqlDataReader reader = cmd.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            orders.Add(new Order
                            {
                                OrderId = reader.GetInt32(reader.GetOrdinal("OrderId")),
                                OrderDate = reader.GetDateTime(reader.GetOrdinal("OrderDate")),
                                Cost = reader.GetDecimal(reader.GetOrdinal("Cost")),
                                Profit = reader.GetDecimal(reader.GetOrdinal("Profit")),
                                NoOfProduct = reader.GetInt32(reader.GetOrdinal("NoOfProduct")),
                                TotalAmount = reader.GetDecimal(reader.GetOrdinal("TotalAmount")),
                                Status = reader.GetString(reader.GetOrdinal("Status"))
                            });
                        }
                    }
                }
            }
            return View(orders);
        }

        // GET: Order/Create
        public IActionResult Create()
        {
            // Pre-fill OrderDate with today's date
            var order = new Order
            {
                OrderDate = DateTime.Today
            };
            return View(order);
        }

        // POST: Order/Create
        [HttpPost]
        [ValidateAntiForgeryToken]
        [Obsolete]
        public IActionResult Create(Order order)
        {
            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                using (SqlCommand cmd = new SqlCommand("UpsertOrder", con))
                {
                    cmd.CommandType = CommandType.StoredProcedure;
                    cmd.Parameters.AddWithValue("@OrderId", 0); // for insert
                    cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
                    cmd.Parameters.AddWithValue("@Cost", order.Cost);
                    cmd.Parameters.AddWithValue("@Profit", order.Profit);
                    cmd.Parameters.AddWithValue("@NoOfProduct", order.NoOfProduct);
                    cmd.Parameters.AddWithValue("@TotalAmount", order.TotalAmount);
                    cmd.Parameters.AddWithValue("@Status", order.Status);
                    cmd.ExecuteNonQuery();
                }
            }
            return RedirectToAction("Index");
        }

        // GET: Order/Edit/5
        [Obsolete]
        public IActionResult Edit(int id)
        {
            Order order = null!;
            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                using (SqlCommand cmd = new SqlCommand("SELECT * FROM Orders WHERE OrderId = @OrderId", con))
                {
                    cmd.Parameters.AddWithValue("@OrderId", id);
                    using (SqlDataReader reader = cmd.ExecuteReader())
                    {
                        if (reader.Read())
                        {
                            order = new Order
                            {
                                OrderId = reader.GetInt32(reader.GetOrdinal("OrderId")),
                                OrderDate = reader.GetDateTime(reader.GetOrdinal("OrderDate")),
                                Cost = reader.GetDecimal(reader.GetOrdinal("Cost")),
                                Profit = reader.GetDecimal(reader.GetOrdinal("Profit")),
                                NoOfProduct = reader.GetInt32(reader.GetOrdinal("NoOfProduct")),
                                TotalAmount = reader.GetDecimal(reader.GetOrdinal("TotalAmount")),
                                Status = reader.GetString(reader.GetOrdinal("Status"))
                            };
                        }
                    }
                }
            }
            return View(order);
        }

        // POST: Order/Edit
        [HttpPost]
        [ValidateAntiForgeryToken]
        [Obsolete]
        public IActionResult Edit(Order order)
        {
            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                using (SqlCommand cmd = new SqlCommand("UpsertOrder", con))
                {
                    cmd.CommandType = CommandType.StoredProcedure;
                    cmd.Parameters.AddWithValue("@OrderId", order.OrderId); // use actual ID
                    cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
                    cmd.Parameters.AddWithValue("@Cost", order.Cost);
                    cmd.Parameters.AddWithValue("@Profit", order.Profit);
                    cmd.Parameters.AddWithValue("@NoOfProduct", order.NoOfProduct);
                    cmd.Parameters.AddWithValue("@TotalAmount", order.TotalAmount);
                    cmd.Parameters.AddWithValue("@Status", order.Status);
                    cmd.ExecuteNonQuery();
                }
            }
            return RedirectToAction("Index");
        }
    }
}
