@model YourNamespace.Models.OrderViewModel
@{
    ViewBag.Title = "Create Order";
}

<h2>Create Order</h2>

<form asp-action="Create" method="post">
    <div class="mb-3">
        <label class="form-label">Order Name</label>
        <input asp-for="OrderName" class="form-control" />
        <span asp-validation-for="OrderName" class="text-danger"></span>
    </div>

    <h4>Customers</h4>
    <div id="customerRepeater">
        @for (int i = 0; i < Model.CustomerOrders.Count; i++)
        {
            <div class="customer-entry mb-2">
                <select class="form-select customer-select" name="CustomerOrders[@i].CustomerId">
                    <option value="">-- Select Customer --</option>
                    @foreach (var c in ViewBag.Customers as List<Customer>)
                    {
                        <option value="@c.CustomerId"
                                @(Model.CustomerOrders[i].CustomerId == c.CustomerId ? "selected=\"selected\"" : "")>
                            @c.Name
                        </option>
                    }
                </select>

                <input name="CustomerOrders[@i].Amount" type="number"
                       value="@Model.CustomerOrders[i].Amount"
                       placeholder="Amount"
                       class="form-control d-inline-block mx-1"
                       style="width: 150px;" />

                <input name="CustomerOrders[@i].IsPaid" type="checkbox"
                       class="form-check-input"
                       @(Model.CustomerOrders[i].IsPaid ? "checked" : "") />

                <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
            </div>
        }
    </div>

    <div class="mt-3">
        <button type="submit" class="btn btn-primary">Save</button>
        <a asp-action="Index" class="btn btn-secondary">Cancel</a>
    </div>
</form>

@section Scripts {
    @{ await Html.RenderPartialAsync("_ValidationScriptsPartial"); }
}
