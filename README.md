@section Scripts {
    <script>
        let index = 0;

        function getCustomerEntryHtml(i) {
            return `
                <div class="customer-entry mb-2">
                    <select class="form-select customer-select" name="CustomerOrders[${i}].CustomerId">
                        <option value="">-- Select Customer --</option>
                        ${customerOptionsHtml}
                    </select>

                    <input name="CustomerOrders[${i}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1 mt-3" style="width: 600px;" />

                    <label class="form-check-label mt-3">
                        <input name="CustomerOrders[${i}].IsPaid" type="checkbox" class="form-check-input" />
                        Is Paid
                    </label>

                    <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                </div>
            `;
        }

        $(document).ready(function () {
            // Add initial customer block
            $('#customerRepeater').html(getCustomerEntryHtml(index));
            index++;

            $('#addCustomer').on('click', function () {
                $('#customerRepeater').append(getCustomerEntryHtml(index));
                index++;
            });

            $(document).on('click', '.remove-entry', function () {
                $(this).closest('.customer-entry').remove();
                // Re-index all existing entries to keep model binding valid
                $('#customerRepeater .customer-entry').each(function (i, el) {
                    $(el).find('select, input').each(function () {
                        const name = $(this).attr('name');
                        if (name) {
                            const updatedName = name.replace(/\CustomerOrders\[\d+\]/, `CustomerOrders[${i}]`);
                            $(this).attr('name', updatedName);
                        }
                    });
                });
                index = $('#customerRepeater .customer-entry').length;
            });

            $('.customer-select').select2(); // if using select2
        });
    </script>
}
