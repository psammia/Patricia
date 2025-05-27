@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Edit Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
    var customerOptionsHtml = new System.Text.StringBuilder();
    foreach (var customer in customers)
    {
        customerOptionsHtml.Append($"<option value=\"{customer.CustomerId}\">{customer.Name}</option>");
    }
}

<h2>Edit Order</h2>

<form asp-action="Edit" method="post">
    <input type="hidden" asp-for="OrderId" />

    <div class="mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" type="date" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="TotalAmount" class="form-label"></label>
        <input asp-for="TotalAmount" class="form-control" />
        <span asp-validation-for="TotalAmount" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Profit" class="form-label"></label>
        <input asp-for="Profit" class="form-control" />
        <span asp-validation-for="Profit" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="NoOfProduct" class="form-label"></label>
        <input asp-for="NoOfProduct" class="form-control" />
        <span asp-validation-for="NoOfProduct" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label>Status</label>
        <select asp-for="StatusCode" class="form-select">
            <option value="">-- Select Status --</option>
            @foreach (var status in (IEnumerable<Status>)ViewBag.Statuses)
            {
                <option value="@status.StatusCode" @(Model.StatusCode == status.StatusCode ? "selected" : "")>@status.StatusCode</option>
            }
        </select>
        <span asp-validation-for="StatusCode" class="text-danger"></span>
    </div>

    <h4>Customers</h4>
    <div id="customerRepeater" class="mb-3">
        @for (int i = 0; i < Model.CustomerOrders.Count; i++)
        {
            var co = Model.CustomerOrders[i];
            <div class="customer-entry mb-3 border p-3 rounded">
                <label>Customer</label>
                <select class="form-select" name="CustomerOrders[@i].CustomerId" required>
                    <option value="">-- Select Customer --</option>
                    @foreach (var customer in customers)
                    {
                        <option value="@customer.CustomerId" @(co.CustomerId == customer.CustomerId ? "selected" : "")>@customer.Name</option>
                    }
                </select>

                <label class="mt-2">Amount</label>
                <input name="CustomerOrders[@i].Amount" class="form-control" value="@co.Amount" required />

                <label class="mt-2">No. of Product</label>
                <input name="CustomerOrders[@i].NoOfProductperCustomer" type="number" class="form-control" value="@co.NoOfProductperCustomer" required />

                <div class="form-check mt-2">
                    <input type="hidden" name="CustomerOrders[@i].IsPaid" value="false" />
                    <input name="CustomerOrders[@i].IsPaid" type="checkbox" value="true" class="form-check-input" @(co.IsPaid ? "checked" : "") />
                    <label class="form-check-label">Is Paid</label>
                </div>

                <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
            </div>
        }
    </div>

    <button type="button" id="addCustomer" class="btn btn-success mb-3">Add Customer</button>

    <div class="form-group">
        <input type="submit" value="Save" class="btn btn-primary" />
    </div>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = @Model.CustomerOrders.Count;
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
}
