        public async Task AddOrderWithCustomersAsync(Order order)
        {
            using var conn = Connection;
            conn.Open();
            using var tran = conn.BeginTransaction();

            try
            {
                var orderId = await conn.ExecuteScalarAsync<int>(
                    "INSERT INTO Orders (OrderDate, Cost, Profit, NoOfProduct, TotalAmount, StatusCode) OUTPUT INSERTED.OrderId VALUES (@OrderDate, @Cost, @Profit, @NoOfProduct, @TotalAmount, @StatusCode)",
                    order, tran);

                foreach (var co in order.CustomerOrders)
                {
                    await conn.ExecuteAsync(
                        "INSERT INTO CustomerOrders (OrderId, CustomerId, Amount, IsPaid) VALUES (@OrderId, @CustomerId, @Amount, @IsPaid)",
                        new { OrderId = orderId, CustomerId = co.CustomerId, Amount = co.Amount, IsPaid = co.IsPaid }, tran);
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
                await conn.ExecuteAsync("UPDATE Orders SET OrderDate = @OrderDate, Cost = @Cost, Profit = @Profit, NoOfProduct=@NoOfProduct, TotalAmount=@TotalAmount, StatusCode=@StatusCode WHERE OrderId = @OrderId", order, tran);

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
