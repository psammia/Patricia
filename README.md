@model OrdersTracking.Models.Order
@{
    var customers = ViewBag.Customers as List<Customer> ?? new();
    var statuses = ViewBag.Statuses as List<Status> ?? new();
}

<h2>Edit Order</h2>

<form asp-action="Edit" method="post">
    @Html.AntiForgeryToken()

    <input asp-for="OrderId" type="hidden" />
    <input type="hidden" name="CustomerOrders.Count" value="@Model.CustomerOrders.Count" />

    <div class="mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" type="date" />
    </div>

    <div class="mb-3">
        <label asp-for="TotalAmount" class="form-label"></label>
        <input asp-for="TotalAmount" class="form-control" />
    </div>

    <div class="mb-3">
        <label asp-for="Profit" class="form-label"></label>
        <input asp-for="Profit" class="form-control" />
    </div>

    <div class="mb-3">
        <label asp-for="NoOfProduct" class="form-label"></label>
        <input asp-for="NoOfProduct" class="form-control" />
    </div>

    <div class="mb-3">
        <label>Status</label>
        <select asp-for="StatusCode" class="form-select" asp-items="@(new SelectList(statuses, "StatusCode", "StatusCode"))">
            <option value="">-- Select Status --</option>
        </select>
    </div>

    <h4>Customer Orders</h4>
    <div id="customerRepeater">
        @for (int i = 0; i < Model.CustomerOrders.Count; i++)
        {
            <div class="border p-3 mb-2 rounded customer-entry">
                <div class="mb-2">
                    <label>Customer</label>
                    <select asp-for="CustomerOrders[@i].CustomerId" class="form-select"
                            asp-items="@(new SelectList(customers, "CustomerId", "Name", Model.CustomerOrders[i].CustomerId))">
                        <option value="">-- Select Customer --</option>
                    </select>
                </div>

                <div class="mb-2">
                    <label>Amount</label>
                    <input asp-for="CustomerOrders[@i].Amount" class="form-control" />
                </div>

                <div class="mb-2">
                    <label>No. of Products</label>
                    <input asp-for="CustomerOrders[@i].NoOfProductperCustomer" class="form-control" type="number" />
                </div>

                <div class="form-check mb-2">
                    <input type="hidden" name="CustomerOrders[@i].IsPaid" value="false" />
                    <input asp-for="CustomerOrders[@i].IsPaid" type="checkbox" class="form-check-input" />
                    <label class="form-check-label" asp-for="CustomerOrders[@i].IsPaid">Is Paid</label>
                </div>

                <button type="button" class="btn btn-danger btn-sm remove-entry">Remove</button>
            </div>
        }
    </div>

    <button type="button" id="addCustomer" class="btn btn-success mb-3">Add Customer</button>

    <input type="submit" value="Save" class="btn btn-primary" />
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = @Model.CustomerOrders.Count;
        const customerOptions = `@Html.Raw(string.Join("", customers.Select(c => $"<option value='{c.CustomerId}'>{c.Name}</option>")))`;

        function getCustomerHtml(i) {
            return `
                <div class="border p-3 mb-2 rounded customer-entry">
                    <div>
                        <label>Customer</label>
                        <select class="form-select" name="CustomerOrders[${i}].CustomerId" required>
                            <option value="">-- Select Customer --</option>
                            ${customerOptions}
                        </select>
                    </div>
                    <div>
                        <label>Amount</label>
                        <input class="form-control" name="CustomerOrders[${i}].Amount" required />
                    </div>
                    <div>
                        <label>No. of Products</label>
                        <input class="form-control" name="CustomerOrders[${i}].NoOfProductperCustomer" type="number" required />
                    </div>
                    <div class="form-check">
                        <input type="hidden" name="CustomerOrders[${i}].IsPaid" value="false" />
                        <input type="checkbox" class="form-check-input" name="CustomerOrders[${i}].IsPaid" value="true" />
                        <label class="form-check-label">Is Paid</label>
                    </div>
                    <button type="button" class="btn btn-danger btn-sm remove-entry">Remove</button>
                </div>`;
        }

        document.getElementById("addCustomer").addEventListener("click", () => {
            document.getElementById("customerRepeater").insertAdjacentHTML("beforeend", getCustomerHtml(index));
            index++;
        });

        document.getElementById("customerRepeater").addEventListener("click", function (e) {
            if (e.target.classList.contains("remove-entry")) {
                e.target.closest(".customer-entry").remove();
            }
        });
    </script>
}
