using System;
using System.Collections.Generic;
using System.Data;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Data.SqlClient;
using OrdersTracking.Models;

namespace OrdersTracking.Controllers
{
    public class OrderController : Controller
    {
        private readonly IConfiguration _config;
        private readonly string _connectionString;

        public OrderController(IConfiguration config)
        {
            _config = config;
            _connectionString = _config.GetConnectionString("DefaultConnection");
        }

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
                                CustomerName = reader.GetString(reader.GetOrdinal("CustomerName")),
                                OrderDate = reader.GetDateTime(reader.GetOrdinal("OrderDate")),
                                TotalAmount = reader.GetDecimal(reader.GetOrdinal("TotalAmount"))
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
        public IActionResult Create(Order order)
        {
            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                using (SqlCommand cmd = new SqlCommand("UpsertOrder", con))
                {
                    cmd.CommandType = CommandType.StoredProcedure;
                    // Pass 0 (or null) for new Order to let the stored procedure insert it
                    cmd.Parameters.AddWithValue("@OrderId", 0);
                    cmd.Parameters.AddWithValue("@CustomerName", order.CustomerName);
                    cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
                    cmd.Parameters.AddWithValue("@TotalAmount", order.TotalAmount);
                    cmd.ExecuteNonQuery();
                }
            }
            return RedirectToAction("Index");
        }

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
                            order.CustomerName = reader.GetString(reader.GetOrdinal("CustomerName"));
                            order.OrderDate = reader.GetDateTime(reader.GetOrdinal("OrderDate"));
                            order.TotalAmount = reader.GetDecimal(reader.GetOrdinal("TotalAmount"));
                        }
                    }
                }
            }
            return View(order);
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public IActionResult Edit(Order order)
        {
            using (SqlConnection con = new SqlConnection(_connectionString))
            {
                con.Open();
                using (SqlCommand cmd = new SqlCommand("UpsertOrder", con))
                {
                    cmd.CommandType = CommandType.StoredProcedure;
                    cmd.Parameters.AddWithValue("@OrderId", order.OrderId);
                    cmd.Parameters.AddWithValue("@CustomerName", order.CustomerName);
                    cmd.Parameters.AddWithValue("@OrderDate", order.OrderDate);
                    cmd.Parameters.AddWithValue("@TotalAmount", order.TotalAmount);
                    cmd.ExecuteNonQuery();
                }
            }
            return RedirectToAction("Index");
        }
    }
}
