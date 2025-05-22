In your _Layout.cshtml, add this in the <head> section or before </body>:

<!-- SweetAlert2 CDN -->
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>


2. Update Index.cshtml (Customer or Order)
Here’s how to add a delete button that uses SweetAlert for confirmation.

Example (in Views/Customer/Index.cshtml):
@foreach (var customer in Model)
{
    <tr>
        <td>@customer.CustomerId</td>
        <td>@customer.Name</td>
        <td>
            <a asp-action="Edit" asp-route-id="@customer.CustomerId" class="btn btn-primary btn-sm">Edit</a>
            <button class="btn btn-danger btn-sm" onclick="confirmDelete(@customer.CustomerId)">Delete</button>
        </td>
    </tr>
}


 3. Add JavaScript for SweetAlert
At the bottom of the page (or in a section like @section Scripts):

<script>
    function confirmDelete(customerId) {
        Swal.fire({
            title: 'Are you sure?',
            text: "Do you really want to delete this customer?",
            icon: 'warning',
            showCancelButton: true,
            confirmButtonColor: '#d33',
            cancelButtonColor: '#3085d6',
            confirmButtonText: 'Yes, delete it!',
        }).then((result) => {
            if (result.isConfirmed) {
                // Redirect to the delete action
                window.location.href = '/Customer/DeleteConfirmed/' + customerId;
            }
        });
    }
</script>


4. Update the Controller for GET-only Deletion (optional)
In your CustomerController.cs, you can simplify by removing the [HttpPost] if you're just using a GET action for deletion:

// GET: /Customer/DeleteConfirmed/5
public async Task<IActionResult> DeleteConfirmed(int id)
{
    await _repo.DeleteCustomerAsync(id);
    return RedirectToAction("Index");
}

Do the same for OrderController if needed.

 For Orders
Just replace customerId with orderId and use:
window.location.href = '/Order/DeleteConfirmed/' + orderId;


