@model OrderTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
}

<h2>Create New Order</h2>

<form asp-action="Create" method="post">

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
    <div class="form-group mb-3">
        <label asp-for="NoOfProduct" class="form-label"></label>
        <input asp-for="NoOfProduct" class="form-control" />
        <span asp-validation-for="NoOfProduct" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="Cost" class="form-label"></label>
        <input asp-for="Cost" class="form-control" />
        <span asp-validation-for="Cost" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="Profit" class="form-label"></label>
        <input asp-for="Profit" class="form-control" />
        <span asp-validation-for="Profit" class="text-danger"></span>
    </div>
    <div class="form-group mb-3">
        <label asp-for="Status" class="form-label"></label>
        <input asp-for="Status" class="form-control" />
        <span asp-validation-for="Status" class="text-danger"></span>
    </div>
    <button type="submit" class="btn btn-success">Save</button>
    <a asp-action="Index" class="btn btn-secondary">Cancel</a>
</form>
