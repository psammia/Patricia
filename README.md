<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>@ViewData["Title"] - OrdersTracking</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    
    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" />

    <!-- DataTables CSS -->
    <link href="https://cdn.datatables.net/1.13.6/css/jquery.dataTables.min.css" rel="stylesheet" />

    <!-- jQuery -->
    <script src="https://code.jquery.com/jquery-3.6.4.min.js"></script>

    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>

    <!-- DataTables JS -->
    <script src="https://cdn.datatables.net/1.13.6/js/jquery.dataTables.min.js"></script>

    <!-- jQuery Validation -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery-validate/1.19.5/jquery.validate.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery-validation-unobtrusive/4.0.0/jquery.validate.unobtrusive.min.js"></script>

    @RenderSection("Scripts", required: false)
</head>
<body>
    <div class="container mt-4">
        @RenderBody()
    </div>
</body>
</html>


@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<OrdersTracking.Models.Customer> ?? new List<OrdersTracking.Models.Customer>();
}

<h2>Create Order</h2>

<form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")" method="post" class="needs-validation" novalidate>
    <div class="mb-3">
        <label asp-for="OrderDate" class="form-label"></label>
        <input asp-for="OrderDate" class="form-control" />
        <span asp-validation-for="OrderDate" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Cost" class="form-label"></label>
        <input asp-for="Cost" class="form-control" />
        <span asp-validation-for="Cost" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Profit" class="form-label"></label>
        <input asp-for="Profit" class="form-control" />
        <span asp-validation-for="Profit" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="NoOfProduct" class="form-label"></label>
        <input asp-for="NoOfProduct" class="form-control" />
        <span asp-validation-for="NoOfProduct" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="TotalAmount" class="form-label"></label>
        <input asp-for="TotalAmount" class="form-control" />
        <span asp-validation-for="TotalAmount" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="StatusCode" class="form-label"></label>
        <input asp-for="StatusCode" class="form-control" />
        <span asp-validation-for="StatusCode" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label class="form-label">Select Customers</label>
        @foreach (var customer in customers)
        {
            <div class="form-check">
                <input class="form-check-input" type="checkbox" name="selectedCustomers" value="@customer.CustomerId" />
                <label class="form-check-label">@customer.Name</label>
            </div>
        }
    </div>

    <button type="submit" class="btn btn-success">Save Order</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}


Index.chstml
@model IEnumerable<OrdersTracking.Models.Order>

@{
    ViewData["Title"] = "Orders List";
}

<h2>Orders</h2>

<a class="btn btn-primary mb-2" asp-action="Create">+ New Order</a>

<table id="ordersTable" class="table table-bordered table-striped">
    <thead>
        <tr>
            <th>Order Date</th>
            <th>Cost</th>
            <th>Profit</th>
            <th>No. of Products</th>
            <th>Total</th>
            <th>Status</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
    @foreach (var item in Model)
    {
        <tr>
            <td>@item.OrderDate.ToShortDateString()</td>
            <td>@item.Cost</td>
            <td>@item.Profit</td>
            <td>@item.NoOfProduct</td>
            <td>@item.TotalAmount</td>
            <td>@item.StatusCode</td>
            <td>
                <a asp-action="Edit" asp-route-id="@item.OrderId" class="btn btn-sm btn-warning">Edit</a>
                <a asp-action="Delete" asp-route-id="@item.OrderId" class="btn btn-sm btn-danger">Delete</a>
            </td>
        </tr>
    }
    </tbody>
</table>

@section Scripts {
    <script>
        $(document).ready(function () {
            $('#ordersTable').DataTable();
        });
    </script>
}

In your Program.cs or Startup.cs, make sure MVC is set up with validation support:
builder.Services.AddControllersWithViews()
    .AddMvcOptions(options => {
        options.ModelBindingMessageProvider.SetValueMustNotBeNullAccessor(
            _ => "This field is required.");
    })
    .AddDataAnnotationsLocalization();

Also ensure _ValidationScriptsPartial.cshtml includes:
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery-validate/1.19.5/jquery.validate.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery-validation-unobtrusive/4.0.0/jquery.validate.unobtrusive.min.js"></script>
