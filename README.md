@model Order
@{
    ViewData["Title"] = "Edit Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
}

<h1>Edit Order</h1>

<form asp-action="Edit">
    @Html.AntiForgeryToken()

    <input type="hidden" asp-for="OrderId" />

    <div class="form-group">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>

    <h4>Customers</h4>
    <div id="customerRepeater">
        @for (int i = 0; i < Model.CustomerOrders.Count; i++)
        {
            <div class="customer-entry mb-2">
                <select class="form-select customer-select" name="CustomerOrders[@i].CustomerId">
                    <option value="">-- Select Customer --</option>
                    @foreach (var c in customers)
                    {
                        <option value="@c.CustomerId" @(Model.CustomerOrders[i].CustomerId == c.CustomerId ? "selected" : "")>@c.Name</option>
                    }
                </select>
                <input name="CustomerOrders[@i].Amount" type="number" value="@Model.CustomerOrders[i].Amount" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                <input name="CustomerOrders[@i].IsPaid" type="checkbox" class="form-check-input" @(Model.CustomerOrders[i].IsPaid ? "checked" : "") />
                <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
            </div>
        }
    </div>

    <button type="button" id="addCustomer" class="btn btn-secondary btn-sm mt-2">+ Add Customer</button>

    <div class="form-group mt-3">
        <button type="submit" class="btn btn-primary">Save</button>
        <a asp-action="Index" class="btn btn-secondary">Cancel</a>
    </div>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = @Model.CustomerOrders.Count;

        const customerOptions = `@foreach (var c in customers) {<text><option value="@c.CustomerId">@c.Name</option></text>}`.trim();

        $('#addCustomer').on('click', function () {
            let newEntry = `
                <div class="customer-entry mb-2">
                    <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                        <option value="">-- Select Customer --</option>
                        ${customerOptions}
                    </select>
                    <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                    <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
                    <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                </div>`;
            $('#customerRepeater').append(newEntry);
            index++;
        });

        $(document).on('click', '.remove-entry', function () {
            $(this).closest('.customer-entry').remove();
        });

        $(document).ready(function () {
            $('.customer-select').select2();
        });
    </script>
}
