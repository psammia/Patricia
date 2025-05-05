Index.cshtml – List all customers

@model IEnumerable<OrdersTracking.Models.Customer>
@{
    ViewData["Title"] = "Customers";
}

<h2>Customer List</h2>

<a asp-action="Create" class="btn btn-primary mb-2">Add New Customer</a>

<table class="table table-bordered">
    <thead>
        <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Phone</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var customer in Model)
        {
            <tr>
                <td>@customer.Name</td>
                <td>@customer.Email</td>
                <td>@customer.Phon Create.cshtml – Create a customer</td>
                <td>
                    <a asp-action="Edit" asp-route-id="@customer.CustomerId" class="btn btn-sm btn-warning">Edit</a>
                </td>
            </tr>
        }
    </tbody>
</table>



 Create.cshtml – Create a customer

@model OrdersTracking.Models.Customer
@{
    ViewData["Title"] = "Add Customer";
}

<h2>Create Customer</h2>

<form asp-action="Create" method="post" class="form">
    <div class="form-group">
        <label asp-for="Name"></label>
        <input asp-for="Name" class="form-control" required />
    </div>
    <div class="form-group">
        <label asp-for="Email"></label>
        <input asp-for="Email" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="Phone"></label>
        <input asp-for="Phone" class="form-control" />
    </div>
    <button type="submit" class="btn btn-success mt-2">Save</button>
    <a asp-action="Index" class="btn btn-secondary mt-2">Cancel</a>
</form>


Edit.cshtml – Edit an existing customer
@model OrdersTracking.Models.Customer
@{
    ViewData["Title"] = "Edit Customer";
}

<h2>Edit Customer</h2>

<form asp-action="Edit" method="post" class="form">
    <input type="hidden" asp-for="CustomerId" />
    <div class="form-group">
        <label asp-for="Name"></label>
        <input asp-for="Name" class="form-control" required />
    </div>
    <div class="form-group">
        <label asp-for="Email"></label>
        <input asp-for="Email" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="Phone"></label>
        <input asp-for="Phone" class="form-control" />
    </div>
    <button type="submit" class="btn btn-success mt-2">Update</button>
    <a asp-action="Index" class="btn btn-secondary mt-2">Cancel</a>
</form>




