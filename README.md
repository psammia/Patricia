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
