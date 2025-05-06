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
                                Status=reader.GetString(reader.GetOrdinal("status")),
                            });
                        }
                    }
                }
            }
            return View(orders);
        }

        public IActionResult Create()
        {
            return View();
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        [Obsolete]
        public IActionResult Create(Order order)
        {


            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                var Order = new Order()
                {
                    OrderDate = DateTime.Today
                };
                using (SqlCommand cmd = new SqlCommand("UpsertOrder", con))
                {
                    cmd.CommandType = CommandType.StoredProcedure;
                    // Pass 0 (or null) for new Order to let the stored procedure insert it
                    cmd.Parameters.AddWithValue("@OrderId", 0);
                    cmd.Parameters.AddWithValue("@OrderDate", DateTime.Today);
                    //cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
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

        [Obsolete]
        public IActionResult Edit(int id)
        {
            var order = new Order();
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
                            order.OrderId = reader.GetInt32(reader.GetOrdinal("OrderId"));
                            order.OrderDate = reader.GetDateTime(reader.GetOrdinal("OrderDate"));
                            order.Cost = reader.GetDecimal(reader.GetOrdinal("Cost"));
                            order.Profit = reader.GetDecimal(reader.GetOrdinal("Profit"));
                            order.NoOfProduct = reader.GetInt32(reader.GetOrdinal("NoOfProduct"));
                            order.TotalAmount = reader.GetDecimal(reader.GetOrdinal("TotalAmount"));
                            order.Status = reader.GetString(reader.GetOrdinal("Status"));
                        }
                    }
                }
            }
            return View(order);
        }

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
                    cmd.Parameters.AddWithValue("@OrderId", 0);
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
