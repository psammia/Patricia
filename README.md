In your CustomerController.cs, make sure you have:

// Shows the Delete confirmation view
public async Task<IActionResult> Delete(int id)
{
    var customer = await _customerRepo.GetCustomerByIdAsync(id);
    if (customer == null)
    {
        return NotFound();
    }
    return View(customer);
}

// POST: Executes the actual deletion
[HttpPost, ActionName("DeleteConfirmed")]
public async Task<IActionResult> DeleteConfirmed(int id)
{
    await _customerRepo.DeleteCustomerAsync(id);
    return RedirectToAction("Index");
}

🔁 The [HttpPost, ActionName("DeleteConfirmed")] attribute means this POST action will match a form with asp-action="DeleteConfirmed".

Ensure your form is posting to DeleteConfirmed (not Delete), like this:
@model Customer

<h2>Delete Customer</h2>

<h4>Are you sure you want to delete @Model.Name?</h4>

<form asp-action="DeleteConfirmed" method="post">
    <input type="hidden" asp-for="CustomerId" />
    <button type="submit" class="btn btn-danger">Yes, Delete</button>
    <a asp-action="Index" class="btn btn-secondary">Cancel</a>
</form>


Make sure your repository has a method like:

ICustomerRepository.cs
Task DeleteCustomerAsync(int customerId);


CustomerRepository.cs
public async Task DeleteCustomerAsync(int customerId)
{
    var sql = "DELETE FROM Customers WHERE CustomerId = @CustomerId";
    await _db.ExecuteAsync(sql, new { CustomerId = customerId });
}


