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
                    <input name="CustomerOrders[${i}].Amount" type="number" step="0.01" class="form-control" required />

                    <label class="mt-2">No. of Product</label>
                    <input name="CustomerOrders[${i}].NoOfProduct" type="number" class="form-control" required />

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
