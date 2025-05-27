@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = 0;
        const customerOptionsHtml = `@Html.Raw(customerOptionsHtml.ToString())`;

        function getCustomerEntryHtml(i) {
            return `
                        <div class="customer-entry mb-3 border p-3 rounded">
                            <label>Customer</label>
                            <select class="form-select customer-select" name="CustomerOrders[${i}].CustomerId" required>
                                <option value="">-- Select Customer --</option>
                                ${customerOptionsHtml}
                            </select>

                            <label class="mt-2">Amount</label>
                            <input name="CustomerOrders[${i}].Amount" class="form-control" required />

                            <label class="mt-2">No.of Product</label>
                                    <input name="CustomerOrders[${i}].NoOfProductperCustomer" type="number" class="form-control" required />

                            <div class="form-check mt-2">
                                <input type="hidden" name="CustomerOrders[${i}].IsPaid" value="false" />
                                <input name="CustomerOrders[${i}].IsPaid" type="checkbox" value="true" class="form-check-input" />
                                <label class="form-check-label">Is Paid</label>
                            </div>

                            <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
                        </div>
                    `;
        }

        $(document).ready(function () {
            $('#customerRepeater').append(getCustomerEntryHtml(index++));
            $('#addCustomer').on('click', function () {
                $('#customerRepeater').append(getCustomerEntryHtml(index++));
            });

            $(document).on('click', '.remove-entry', function () {
                $(this).closest('.customer-entry').remove();
                // Re-index names
                $('#customerRepeater .customer-entry').each(function (i, el) {
                    $(el).find('select, input').each(function () {
                        const oldName = $(this).attr('name');
                        if (oldName) {
                            const newName = oldName.replace(/CustomerOrders\[\d+\]/, `CustomerOrders[${i}]`);
                            $(this).attr('name', newName);
                        }
                    });
                });
                index = $('#customerRepeater .customer-entry').length;
            });
        });
    </script>

    orderRepository
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

                //foreach (var coo in order.CustomerOrders)
                //{
                //    Console.WriteLine($"CustomerId: {coo.CustomerId},Amount: {coo.Amount}, IsPaid: {coo.IsPaid}, NoOfProductPerclient:{coo.NoOfProductperCustomer}");
                //}

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
