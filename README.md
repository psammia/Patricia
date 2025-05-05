index:
@model IEnumerable<OrdersTracking.Models.Order>

@{
    ViewData["Title"] = "Orders List";
}

<h2>Orders List</h2>

<a asp-action="Create" class="btn btn-primary mb-3">Create New Order</a>

<table class="table table-striped">
    <thead>
        <tr>
            <th>Order ID</th>
            <th>Customer Name</th>
            <th>Order Date</th>
            <th>Total Amount</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        @foreach (var order in Model)
        {
            <tr>
                <td>@order.OrderId</td>
                <td>@order.CustomerName</td>
                <td>@order.OrderDate.ToShortDateString()</td>
                <td>@order.TotalAmount.ToString("C")</td>
                <td>
                    <a asp-action="Edit" asp-route-id="@order.OrderId" class="btn btn-sm btn-warning">Edit</a>
                </td>
            </tr>
        }
    </tbody>
</table>


Create:
@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
}

<h2>Create New Order</h2>

<form asp-action="Create" method="post">
    <div class="form-group mb-3">
        <label asp-for="CustomerName" class="form-label"></label>
        <input asp-for="CustomerName" class="form-control" />
        <span asp-validation-for="CustomerName" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" type="date" class="form-control" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="TotalAmount" class="form-label"></label>
        <input asp-for="TotalAmount" class="form-control" />
        <span asp-validation-for="TotalAmount" class="text-danger"></span>
    </div>
    <button type="submit" class="btn btn-success">Save</button>
    <a asp-action="Index" class="btn btn-secondary">Cancel</a>
</form>


Index:
@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Edit Order";
}

<h2>Edit Order</h2>

<form asp-action="Edit" method="post">
    <input type="hidden" asp-for="OrderId" />
    <div class="form-group mb-3">
        <label asp-for="CustomerName" class="form-label"></label>
        <input asp-for="CustomerName" class="form-control" />
        <span asp-validation-for="CustomerName" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" type="date" class="form-control" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="TotalAmount" class="form-label"></label>
        <input asp-for="TotalAmount" class="form-control" />
        <span asp-validation-for="TotalAmount" class="text-danger"></span>
    </div>
    <button type="submit" class="btn btn-primary">Update</button>
    <a asp-action="Index" class="btn btn-secondary">Cancel</a>
</form>
