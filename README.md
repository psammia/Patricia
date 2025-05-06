
<div class="form-group">
    <label asp-for="OrderDate" class="control-label"></label>
    <input asp-for="OrderDate" class="form-control" type="date" />
    <span asp-validation-for="OrderDate" class="text-danger"></span>
</div>



<div class="form-group">
    @Html.LabelFor(model => model.OrderDate, htmlAttributes: new { @class = "control-label" })
    @Html.TextBoxFor(model => model.OrderDate, "{0:yyyy-MM-dd}", new { @class = "form-control", type = "date" })
    @Html.ValidationMessageFor(model => model.OrderDate, "", new { @class = "text-danger" })
</div>