<label>Order Date</label>
<input asp-for="OrderDate" type="date" class="form-control" value="@DateTime.Today.ToString("yyyy-MM-dd")" /><br />
<span asp-validation-for="OrderDate" class="text-danger"></span>
