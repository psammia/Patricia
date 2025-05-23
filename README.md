@model Order
@{
    var customers = ViewBag.AllCustomers as List<Customer>;
    var statuses = ViewBag.Statuses as List<Status>;
}



Include Select2 and jQuery in your layout if not already:
<link href="https://cdn.jsdelivr.net/npm/select2@4.1.0/dist/css/select2.min.css" rel="stylesheet" />
<script src="https://cdn.jsdelivr.net/npm/select2@4.1.0/dist/js/select2.min.js"></script>


Then the repeater UI:
<form asp-action="Create" method="post">
    <!-- Order fields -->
    <div class="mb-3">
        <label asp-for="OrderDate">Order Date</label>
        <input asp-for="OrderDate" type="date" class="form-control" value="@DateTime.Today.ToString("yyyy-MM-dd")" />
    </div>

    <!-- Other fields omitted for brevity -->

    <!-- Status dropdown -->
    <div class="mb-3">
        <label>Status</label>
        <select asp-for="StatusCode" class="form-select">
            <option value="">-- Select Status --</option>
            @foreach (var status in statuses)
            {
                <option value="@status.Code">@status.Code</option>
            }
        </select>
    </div>

    <!-- Customer + Amount + Paid Repeater -->
    <div id="customerRepeater">
        <div class="customer-entry mb-2">
            <select class="form-select customer-select" name="CustomerOrders[0].CustomerId">
                <option value="">-- Select Customer --</option>
                @foreach (var c in customers)
                {
                    <option value="@c.CustomerId">@c.Name</option>
                }
            </select>
            <input name="CustomerOrders[0].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
            <input name="CustomerOrders[0].IsPaid" type="checkbox" class="form-check-input" />
            <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
        </div>
    </div>
    <button type="button" class="btn btn-secondary" id="addCustomer">Add Another Customer</button>

    <div class="mt-3 text-end">
        <button type="submit" class="btn btn-primary">Save Order</button>
    </div>
</form>

<script>
    let index = 1;
    $('#addCustomer').on('click', function () {
        let newEntry = `
        <div class="customer-entry mb-2">
            <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                <option value="">-- Select Customer --</option>
                ${@Html.Raw(Newtonsoft.Json.JsonConvert.SerializeObject(customers))
                    .Replace("\"", "\\\"")
                    .Replace("{", "{ ")
                    .Replace("}", " }")
                    .Replace(",", ", ")}
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


