public async Task AddOrderWithCustomersAsync(Order order)
{
    using var connection = new SqlConnection(_connectionString);
    connection.Open();
    using var transaction = connection.BeginTransaction();

    try
    {
        var orderId = await connection.ExecuteScalarAsync<int>(
            "INSERT INTO Orders (OrderDate, Cost, Profit, NoOfProduct, TotalAmount, StatusCode) OUTPUT INSERTED.OrderId VALUES (@OrderDate, @Cost, @Profit, @NoOfProduct, @TotalAmount, @StatusCode)",
            order, transaction);

        foreach (var co in order.CustomerOrders)
        {
            await connection.ExecuteAsync(
                "INSERT INTO CustomerOrder (OrderId, CustomerId, Amount, IsPaid) VALUES (@OrderId, @CustomerId, @Amount, @IsPaid)",
                new { OrderId = orderId, CustomerId = co.CustomerId, Amount = co.Amount, IsPaid = co.IsPaid }, transaction);
        }

        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}


