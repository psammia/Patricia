_Layout.cshtml

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - OrderTracking</title>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
    <link rel="stylesheet" href="~/lib/DataTables/datatables.min.css" />
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container-fluid">
                <a class="navbar-brand" asp-area="" asp-controller="Home" asp-action="Index">OrderTracking</a>
                <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target=".navbar-collapse" aria-controls="navbarSupportedContent"
                        aria-expanded="false" aria-label="Toggle navigation">
                    <span class="navbar-toggler-icon"></span>
                </button>
                <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
                    <ul class="navbar-nav flex-grow-1">
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Index">Home</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
                        </li>
                    </ul>
                </div>
            </div>
        </nav>
    </header>
    <div class="container">
        <main role="main" class="pb-3">
            @RenderBody()
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; 2025 - OrderTracking - <a asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
        </div>
    </footer>
    <script src="~/lib/jquery/dist/jquery.min.js"></script>
    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/lib/DataTables/datatables.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>
    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>


Index.chtml
@model IEnumerable<OrderTracking.Models.Order>

@{
    ViewData["Title"] = "Orders List";
}



@section Scripts{
<script>
    $(document).ready(function () {
        $('#ordersTable').DataTable();
    });
</script>
}

<h2>Orders List</h2>

<a asp-action="Create" class="btn btn-primary mb-3">Create New Order</a>

<table id="ordersTable" class="table table-striped">
    <thead>
        <tr>
            <th>Order ID</th>
            <th>Order Date</th>
            <th>Total Amount</th>
            <th>No of Products</th>
            <th>Cost</th>
            <th>Profit</th>.
            <th>StatusCode</th>
            <th></th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        @foreach (var order in Model)
        {
            <tr>
                <td>@order.OrderId</td>
                <td>@order.OrderDate.ToShortDateString()</td>
                <td>@order.TotalAmount?.ToString("C")</td>
                <td>@order.NoOfProduct</td>
                <td>@order.Cost?.ToString("C")</td>
                <td>@order.Profit?.ToString("C")</td>
                <td>@order.StatusCode</td>
                <td>
                    <a asp-action="Edit" asp-route-id="@order.OrderId" class="btn btn-sm btn-warning">Edit</a>
                </td>
                <td>
                    <a asp-action="AssignCustomersToOrder" asp-route-id="@order.OrderId" class="btn btn-sm btn-success">Assign Customers</a>
                </td>
            </tr>
        }
    </tbody>
</table>
