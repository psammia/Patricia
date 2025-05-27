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
            <input name="CustomerOrders[@i].Amount" class="form-control" 
                   value="@co.Amount.ToString("0.##")" required />

            <label class="mt-2">No. of Product</label>
            <input name="CustomerOrders[@i].NoOfProductperCustomer" type="number" class="form-control" 
                   value="@co.NoOfProductperCustomer" required />

            <div class="form-check mt-2">
                <input type="hidden" name="CustomerOrders[@i].IsPaid" value="false" />
                <input name="CustomerOrders[@i].IsPaid" type="checkbox" value="true" class="form-check-input"
                       @(co.IsPaid ? "checked" : "") />
                <label class="form-check-label">Is Paid</label>
            </div>

            <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
        </div>
    }
</div>
