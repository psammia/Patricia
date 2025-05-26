@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
    var statuses = ViewBag.Statuses as List<Status> ?? new List<Status>();
}

<h2>Create Order</h2>

<div class="card">
    <div class="card-content collapse show">
        <div class="card-body">
            <form asp-action="@(Model.OrderId == 0 ? "Create" : "Edit")" method="post" class="needs-validation" novalidate>
                <div class="mb-3">
                    <input asp-for="OrderDate" type="date" class="form-control" value="@DateTime.Today.ToString("yyyy-MM-dd")" /><br />
                    <span asp-validation-for="OrderDate" class="text-danger"></span>
                </div>

                <div class="mb-3">
                    <label asp-for="TotalAmount" class="form-label"></label>
                    <input asp-for="TotalAmount" class="form-control" />
                    <span asp-validation-for="TotalAmount" class="text-danger"></span>
                </div>

               @* <div class="mb-3">
                    <label asp-for="Cost" class="form-label"></label>
                    <input asp-for="Cost" class="form-control" />
                    <span asp-validation-for="Cost" class="text-danger"></span>
                </div>*@

                <div class="mb-3">
                    <label asp-for="NoOfProduct" class="form-label"></label>
                    <input asp-for="NoOfProduct" class="form-control" />
                    <span asp-validation-for="NoOfProduct" class="text-danger"></span>
                </div>

                <div class="mb-3">
                    <label asp-for="Profit" class="form-label"></label>
                    <input asp-for="Profit" class="form-control" />
                    <span asp-validation-for="Profit" class="text-danger"></span>
                </div>

                <div class="mb-3">
                    <label asp-for="StatusCode">Status</label>
                    <select asp-for="StatusCode"
                            class="form-control"
                            asp-items="@(new SelectList(ViewBag.Statuses, "StatusCode", "StatusCode"))">
                        <option value="">-- Select Status --</option>
                    </select>
                    <span asp-validation-for="StatusCode" class="text-danger"></span>
                </div>

                <!-- Customer + Amount + Paid Repeater -->
                <div id="customerRepeater">
                    <div class="customer-entry mb-2">
                        <select class="form-select customer-select" name="CustomerOrders[0].CustomerId">
                            <option value="">-- Select Customer --</option>
                            @foreach (var c in customers)
                            {
                                <option value="@c.CustomerId">@c.Name</option>
                            }
                        </select>
                        <input name="CustomerOrders[0].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1 mt-3" style="width: 600px;" />
                        <label for="isPaidCheckbox"> Is Paid</label>
                        <input name="CustomerOrders[0].IsPaid" type="checkbox" class="form-check-input mt-4 m-2" id="isPaidCheckbox" />
                        <button type="button" class="btn btn-danger btn-sm remove-entry">x</button>
                    </div>
                </div>
                <button type="button" class="btn btn-secondary mt-2" id="addCustomer">+ Add Customer</button>

                <button type="submit" class="btn btn-success mt-2">Save Order</button>
            </form>
        </div>
    </div>
</div>

@section Scripts {
    <script>
        let index = 1;

        // Render options once
        const customerOptions = `@foreach (var c in customers)
        {
            <text><option value="@c.CustomerId">@c.Name</option></text>
        }`.trim();

        $('#addCustomer').on('click', function () {
            let newEntry = `
                                               <div class="customer-entry mb-2">
                                                   <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                                                       <option value="">-- Select Customer --</option>
                                                       ${customerOptions}
                                                   </select>
                                                   <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                                                   <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
                                                   <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                                               </div>`;
            $('#customerRepeater').append(newEntry);
            index++;
        });

        $(document).on('click', '.remove-entry', function () {
            $(this).closest('.customer-entry').remove();
        });

        $(document).ready(function () {
            $('.customer-select').select2();
        });
    </script>
}
