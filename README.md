<div class="row">
    <div class="col-xs-12">
        <h3>Dashboard</h3>
        <ol class="breadcrumb sgbl-breadcrumb">
            <li>
                <a href="~/Home/Index/Redirect">Home</a>
            </li>
            <li class="active">Pending Invoices</li>
        </ol>
    </div>
</div>

<div class="card">
    <div class="card-content collapse show">
        <div class="card-body">
            <select id="select-channel" class="form-control">
                <option value="" disabled selected>-- Select Channel --</option>
                @foreach (var item in (ViewBag.Channels as List<SelectListItem> ?? new List<SelectListItem>()))
                {
                    <option value="@item.Value">@item.Text</option>
                }
            </select>

            <div class="row">
                <div class="col-xs-12">
                    <div id="partialContainer"></div>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
    $(document).ready(function () {
        $('#select-channel').on('change', function () {
            var selected = $(this).val();

            const getPendingInvoicesByChannel = {
                channelCode: selected
            }

            if (selected) {
                $.ajax({
                    url: '@Url.Action("GetPendingInvoicesByChannel", "Invoice")',
                    type: 'POST',
                    data: getPendingInvoicesByChannel,
                    success: function (result) {

                        

                        $('#partialContainer').html(result);
                        $("#table-pending-invoices").DataTable({});
                    }
                });

            }
        })
    });

</script>
