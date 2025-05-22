Views/Customer/Index.cshtml
@model IEnumerable<Customer>

@{
    ViewData["Title"] = "Customers";
}

<h2>Customers</h2>

<a href="@Url.Action("Create", "Customer")" class="btn btn-primary mb-3">Create New Customer</a>

<table id="customersTable" class="table table-striped table-bordered">
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
                <td>@customer.Phone</td>
                <td>
                    <a href="@Url.Action("Edit", "Customer", new { id = customer.CustomerId })" class="btn btn-warning btn-sm">Edit</a>
                    <a href="@Url.Action("Delete", "Customer", new { id = customer.CustomerId })" class="btn btn-danger btn-sm">Delete</a>
                </td>
            </tr>
        }
    </tbody>
</table>

@section Scripts {
    <script>
        $(document).ready(function () {
            $('#customersTable').DataTable();
        });
    </script>
}


 Views/Customer/Create.cshtml & Edit.cshtml (shared structure)
Use the same file with conditional logic for edit/create if needed.

@model Customer
@{
    ViewData["Title"] = Model.CustomerId == 0 ? "Create Customer" : "Edit Customer";
}

<h2>@ViewData["Title"]</h2>

<form asp-action="@(Model.CustomerId == 0 ? "Create" : "Edit")" method="post" class="needs-validation" novalidate>
    <div class="mb-3">
        <label asp-for="Name" class="form-label"></label>
        <input asp-for="Name" class="form-control" />
        <span asp-validation-for="Name" class="text-danger"></span>
    </div>
    <div class="mb-3">
        <label asp-for="Email" class="form-label"></label>
        <input asp-for="Email" class="form-control" />
        <span asp-validation-for="Email" class="text-danger"></span>
    </div>
    <div class="mb-3">
        <label asp-for="Phone" class="form-label"></label>
        <input asp-for="Phone" class="form-control" />
        <span asp-validation-for="Phone" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-success">Save</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}


Views/Customer/Delete.cshtml
@model Customer
@{
    ViewData["Title"] = "Delete Customer";
}

<h2>Delete Customer</h2>

<h4>Are you sure you want to delete @Model.Name?</h4>

<form asp-action="DeleteConfirmed" method="post">
    <input type="hidden" asp-for="CustomerId" />
    <button type="submit" class="btn btn-danger">Yes, Delete</button>
    <a asp-action="Index" class="btn btn-secondary">Cancel</a>
</form>


