@for (int i = 0; i < Model.CustomerOrders.Count; i++)
{
    foreach (var c in ViewBag.Customers as List<Customer>)
    {
        <option value="@c.CustomerId" @(Model.CustomerOrders[i].CustomerId == c.CustomerId ? "selected=\"selected\"" : "")>
            @c.Name
        </option>
    }
}
