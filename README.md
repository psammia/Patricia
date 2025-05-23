
    <h4>Customers</h4>
    <div id="customerRepeater">
        @for (int i = 0; i < Model.CustomerOrders.Count; i++)
        {
            <div class="customer-entry mb-2">
                <select class="form-select customer-select" name="CustomerOrders[@i].CustomerId">
                    <option value="">-- Select Customer --</option>
                    @for (int i = 0; i < Model.CustomerOrders.Count; i++)
                    {
                        foreach (var c in ViewBag.Customers as List<Customer>)
                        {
                            <option value="@c.CustomerId" @(Model.CustomerOrders[i].CustomerId == c.CustomerId ? "selected=\"selected\"" : "")>
                                @c.Name
                            </option>
                        }
                    }
                </select>
                <input name="CustomerOrders[@i].Amount" type="number" value="@Model.CustomerOrders[i].Amount" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                <input name="CustomerOrders[@i].IsPaid" type="checkbox" class="form-check-input" @(Model.CustomerOrders[i].IsPaid ? "checked" : "") />
                <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
            </div>
        }
    </div>
