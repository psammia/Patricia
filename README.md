@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
    var statuses = ViewBag.Statuses as List<Status> ?? new List<Status>();
}

<h2>Create Order</h2>

<div class="card">
    <div class="card-content collapse show">
        <div class="card-body">
            <form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")" method="post" class="needs-validation" novalidate>
                @* Date *@
                <div class="mb-3">
                    <label asp-for="OrderDate" class="form-label"></label>
                    <input asp-for="OrderDate" type="date" class="form-control" value="@DateTime.Today.ToString("yyyy-MM-dd")" />
                    <span asp-validation-for="OrderDate" class="text-danger"></span>
                </div>

                @* TotalAmount *@
                <div class="mb-3">
                    <label asp-for="TotalAmount" class="form-label"></label>
                    <input asp-for="TotalAmount" class="form-control" />
                    <span asp-validation-for="TotalAmount" class="text-danger"></span>
                </div>

                @* NoOfProduct *@
                <div class="mb-3">
                    <label asp-for="NoOfProduct" class="form-label"></label>
                    <input asp-for="NoOfProduct" class="form-control" />
                    <span asp-validation-for="NoOfProduct" class="text-danger"></span>
                </div>

                @* Profit *@
                <div class="mb-3">
                    <label asp-for="Profit" class="form-label"></label>
                    <input asp-for="Profit" class="form-control" />
                    <span asp-validation-for="Profit" class="text-danger"></span>
                </div>

                @* Status dropdown *@
                <div class="mb-3">
                    <label asp-for="StatusCode" class="form-label">Status</label>
                    <select asp-for="StatusCode" class="form-control" asp-items="@(new SelectList(statuses, "StatusCode", "StatusCode"))">
                        <option value="">-- Select Status --</option>
                    </select>
                    <span asp-validation-for="StatusCode" class="text-danger"></span>
                </div>

                @* Customer/Amount/IsPaid Repeater *@
                <div id="customerRepeater">
                    <div class="customer-entry mb-2">
                        <select class="form-select customer-select" name="CustomerOrders[0].CustomerId">
                            <option value="">-- Select Customer --</option>
                            @foreach (var c in customers)
                            {
                                <option value="@c.CustomerId">@c.Name</option>
                            }
                        </select>

                        <input name="CustomerOrders[0].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1 mt-3" style="width: 600px;" />

                        <label class="form-check-label mt-3">
                            <input name="CustomerOrders[0].IsPaid" type="checkbox" class="form-check-input" />
                            Is Paid
                        </label>

                        <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                    </div>
                </div>

                <button type="button" class="btn btn-secondary mt-2" id="addCustomer">+ Add Customer</button>

                <div class="mt-4">
                    <button type="submit" class="btn btn-success">Save Order</button>
                </div>
            </form>
        </div>
    </div>
</div>

@section Scripts {
    <script>
        let index = 1;

        const customerOptionsHtml = `@Html.Raw(string.Join("", customers.Select(c => $"<option value='{c.CustomerId}'>{c.Name}</option>")))`;

        $('#addCustomer').on('click', function () {
            const entryHtml = `
                <div class="customer-entry mb-2">
                    <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                        <option value="">-- Select Customer --</option>
                        ${customerOptionsHtml}
                    </select>

                    <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1 mt-3" style="width: 600px;" />

                    <label class="form-check-label mt-3">
                        <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
                        Is Paid
                    </label>

                    <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                </div>
            `;
            $('#customerRepeater').append(entryHtml);
            index++;
        });

        $(document).on('click', '.remove-entry', function () {
            $(this).closest('.customer-entry').remove();
        });

        $(document).ready(function () {
            $('.customer-select').select2(); // if using select2
        });
    </script>
}
