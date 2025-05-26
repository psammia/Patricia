@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<SelectListItem>;
    var customerOptionsHtml = new System.Text.StringBuilder();
    foreach (var customer in customers)
    {
        customerOptionsHtml.Append($"<option value=\"{customer.Value}\">{customer.Text}</option>");
    }
}

<h2>Create Order</h2>

<form asp-action="Create" method="post">
    <div class="mb-3">
        <label asp-for="Description" class="form-label"></label>
        <input asp-for="Description" class="form-control" />
        <span asp-validation-for="Description" class="text-danger"></span>
    </div>
    <div class="mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" type="datetime-local" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>
    <div class="mb-3">
        <label asp-for="Cost" class="form-label"></label>
        <input asp-for="Cost" class="form-control" type="number" step="0.01" />
        <span asp-validation-for="Cost" class="text-danger"></span>
    </div>
    <div class="mb-3">
        <label asp-for="Profit" class="form-label"></label>
        <input asp-for="Profit" class="form-control" type="number" step="0.01" />
        <span asp-validation-for="Profit" class="text-danger"></span>
    </div>

    <h4>Customers</h4>
    <div id="customerRepeater" class="mb-3">
        <!-- JavaScript will populate this section -->
    </div>
    <button type="button" id="addCustomer" class="btn btn-success mb-3">Add Customer</button>

    <div class="form-group">
        <input type="submit" value="Create" class="btn btn-primary" />
    </div>
</form>

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
                    <input name="CustomerOrders[${i}].Amount" type="number" step="0.01" class="form-control" required />

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
            // Add first entry on load
            $('#customerRepeater').append(getCustomerEntryHtml(index));
            index++;

            $('#addCustomer').on('click', function () {
                $('#customerRepeater').append(getCustomerEntryHtml(index));
                index++;
            });

            $(document).on('click', '.remove-entry', function () {
                $(this).closest('.customer-entry').remove();

                // Re-index all inputs for proper model binding
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

            // Optional: If using Select2
            // $('.customer-select').select2();
        });
    </script>
}

