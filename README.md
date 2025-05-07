
<div class="row">
    <div class="col-xs-12">
        <h3>Customers List</h3>
        <ol class="breadcrumb sgbl-breadcrumb">
            <li>
                <a href="~/Home/Index/Redirect">Home</a>
            </li>
            <li class="active">Customers List</li>
        </ol>
    </div>
</div>


<div><a asp-action="Create" class="btn btn-primary mb-3">Create New Customer</a></div>


<div class="card">
    <div class="card-content collapse show">
        <div class="card-body">
            <table id="CustomersTable" class="table table-striped">
                <thead>
                    <tr>
                        <th>Name</th>
                        <th>Address</th>
                        <th>Phone</th>
                        <th></th>

                    </tr>
                </thead>
                <tbody>
                    @foreach (var customer in Model)
                    {
                        <tr>
                            <td>@customer.Name</td>
                            <td>@customer.Address</td>
                            <td>@customer.Phone</td>
                            <td>
                                <a asp-action="Edit" asp-route-id="@customer.CustomerId" class="btn btn-sm btn-warning">Edit</a>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    </div>
</div>
