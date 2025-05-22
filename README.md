<label for="StatusCode">Status</label>
<select asp-for="StatusCode" class="form-control" asp-items="@(new SelectList(ViewBag.Statuses, "StatusCode", "StatusCode"))">
    <option value="">-- Select Status --</option>
</select>
<br />

