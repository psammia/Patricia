@{
    ViewData["Title"] = "Home";
}

<div class="text-center">
    <h1 class="display-4">Welcome to Order Tracking System</h1>
    <p class="lead">Use the links below to manage your data.</p>

    <div class="mt-4">
        <a asp-controller="Order" asp-action="Index" class="btn btn-primary btn-lg me-3">Manage Orders</a>
        <a asp-controller="Customer" asp-action="Index" class="btn btn-success btn-lg">Manage Customers</a>
    </div>
</div>


