using OrdersTracking.Models;

public interface ICustomerRepository
{
    Task<IEnumerable<Customer>> GetAllCustomersAsync();
    Task<Customer?> GetCustomerByIdAsync(int id);
    Task AddCustomerAsync(Customer customer);
    Task UpdateCustomerAsync(Customer customer);
    Task DeleteCustomerAsync(int id);
}



using OrdersTracking.Models;

public interface IOrderRepository
{
    Task<IEnumerable<Order>> GetAllOrdersAsync();
    Task<Order?> GetOrderByIdAsync(int id);
    Task AddOrderWithCustomersAsync(Order order, int[] customerIds);
    Task UpdateOrderWithCustomersAsync(Order order, int[] customerIds);
    Task DeleteOrderAsync(int id);
}


using System.Data;
using Dapper;
using Microsoft.Data.SqlClient;
using OrdersTracking.Models;

public class CustomerRepository : ICustomerRepository
{
    private readonly IConfiguration _config;

    public CustomerRepository(IConfiguration config)
    {
        _config = config;
    }

    private IDbConnection Connection => new SqlConnection(_config.GetConnectionString("DefaultConnection"));

    public async Task<IEnumerable<Customer>> GetAllCustomersAsync()
    {
        using var conn = Connection;
        return await conn.QueryAsync<Customer>("SELECT * FROM Customers");
    }

    public async Task<Customer?> GetCustomerByIdAsync(int id)
    {
        using var conn = Connection;
        return await conn.QuerySingleOrDefaultAsync<Customer>("SELECT * FROM Customers WHERE CustomerId = @id", new { id });
    }

    public async Task AddCustomerAsync(Customer customer)
    {
        using var conn = Connection;
        await conn.ExecuteAsync("INSERT INTO Customers (Name) VALUES (@Name)", customer);
    }

    public async Task UpdateCustomerAsync(Customer customer)
    {
        using var conn = Connection;
        await conn.ExecuteAsync("UPDATE Customers SET Name = @Name WHERE CustomerId = @CustomerId", customer);
    }

    public async Task DeleteCustomerAsync(int id)
    {
        using var conn = Connection;
        await conn.ExecuteAsync("DELETE FROM Customers WHERE CustomerId = @id", new { id });
    }
}


using System.Data;
using Dapper;
using Microsoft.Data.SqlClient;
using OrdersTracking.Models;

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

    public async Task AddOrderWithCustomersAsync(Order order, int[] customerIds)
    {
        using var conn = Connection;
        conn.Open();
        using var tran = conn.BeginTransaction();

        try
        {
            var insertOrderQuery = "INSERT INTO Orders (Description, Cost, Profit) VALUES (@Description, @Cost, @Profit); SELECT CAST(SCOPE_IDENTITY() as int)";
            int orderId = await conn.ExecuteScalarAsync<int>(insertOrderQuery, order, tran);

            foreach (var customerId in customerIds)
            {
                await conn.ExecuteAsync("INSERT INTO CustomerOrders (OrderId, CustomerId, IsPaid) VALUES (@OrderId, @CustomerId, 0)",
                    new { OrderId = orderId, CustomerId = customerId }, tran);
            }

            tran.Commit();
        }
        catch
        {
            tran.Rollback();
            throw;
        }
    }

    public async Task UpdateOrderWithCustomersAsync(Order order, int[] customerIds)
    {
        using var conn = Connection;
        conn.Open();
        using var tran = conn.BeginTransaction();

        try
        {
            await conn.ExecuteAsync("UPDATE Orders SET Description = @Description, Cost = @Cost, Profit = @Profit WHERE OrderId = @OrderId", order, tran);

            // Delete previous customer associations
            await conn.ExecuteAsync("DELETE FROM CustomerOrders WHERE OrderId = @OrderId", new { order.OrderId }, tran);

            // Reinsert
            foreach (var customerId in customerIds)
            {
                await conn.ExecuteAsync("INSERT INTO CustomerOrders (OrderId, CustomerId, IsPaid) VALUES (@OrderId, @CustomerId, 0)",
                    new { OrderId = order.OrderId, CustomerId = customerId }, tran);
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
