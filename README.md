✅ Main View: NotifyWarehouse.cshtml
cshtml
Copy
Edit
@{
    ViewBag.Title = "Notify Warehouse";
}

<!-- SweetAlert2 Styles and Script -->
<link href="https://cdn.jsdelivr.net/npm/sweetalert2@11/dist/sweetalert2.min.css" rel="stylesheet" />
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>

<div class="row">
    <div class="col-md-12">
        <h3>Notify Warehouse</h3>
    </div>
    <div class="col-md-12">
        <ol class="breadcrumb sgbl-breadcrumb">
            <li><a href="~/Home/Index/Redirect">Home</a></li>
            <li class="active">List of Boxes to be Sent to Warehouse</li>
        </ol>
    </div>
</div>

<div class="card">
    <div class="card-header"></div>
    <div class="card-content collapse show">
        <div class="card-body card-dashboard">
            <button id="notifyButton" class="btn btn-primary" disabled>Notify Warehouse</button>
        </div>
        <div id="TableDisplay" class="table-spacer"></div>
        <br />
    </div>
</div>

<script>
    $(document).ready(() => {
        loadTable();

        // Notify click handler
        $('#notifyButton').on('click', function () {
            const selectedIds = $('.row-checkbox:checked').map(function () {
                return $(this).val();
            }).get();

            if (selectedIds.length === 0) return;

            Swal.fire({
                title: 'Confirm Notification',
                text: `Notify warehouse for ${selectedIds.length} box(es)?`,
                icon: 'warning',
                showCancelButton: true,
                confirmButtonText: 'Yes, notify',
                cancelButtonText: 'Cancel'
            }).then(result => {
                if (result.isConfirmed) {
                    $.post('/BoxRCA/NotifyWareHouse/', { containerIds: selectedIds.join(',') })
                        .done(() => {
                            Swal.fire('Done!', 'Warehouse has been notified.', 'success');
                            loadTable(); // reload to clear checkboxes
                        })
                        .fail(xhr => $('#MainRenderLocation').html(xhr.responseText));
                }
            });
        });
    });

    function loadTable() {
        $.post('/BoxRCA/GetContainerToNotifyWarehouse/', {}, function (html) {
            $('#TableDisplay').html(html);
        }).fail(xhr => {
            $('#MainRenderLocation').html(xhr.responseText);
        });
    }
</script>
✅ Partial View: GetContainerToNotifyWarehouse.cshtml
cshtml
Copy
Edit
@using Alterna.Archive.Core.Models
@model Alterna.Archive.Core.Models.TableModel.ContainerToNotifyWarehouseTableModel

<table id="TblContainertoNotifyWarehouseTable" class="table table-striped table-bordered" style="width:100%;">
    <thead>
        <tr>
            <th class="text-center"><input type="checkbox" id="checkAllBoxes" /></th>
            <th>Box Ref</th>
            <th>Box Type</th>
            <th>Status Code</th>
            <th>Last Modified By</th>
            <th>Last Modified Date</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var container in Model.ContainersToNotifyWarehouseList)
        {
            <tr>
                <td class="text-center">
                    <input type="checkbox" class="row-checkbox" value="@container.Id" />
                </td>
                <td>@container.Code</td>
                <td>@container.ContainerType</td>
                <td>@container.StatusCode</td>
                <td>@container.LastModifiedBy</td>
                <td>@container.LastModifiedDate.ToString("dd/MM/yyyy")</td>
            </tr>
        }
    </tbody>
</table>

<script>
    $(document).ready(function () {
        const table = $('#TblContainertoNotifyWarehouseTable').DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        $('#checkAllBoxes').on('change', function () {
            $('.row-checkbox').prop('checked', this.checked);
            toggleNotifyButton();
        });

        $(document).on('change', '.row-checkbox', function () {
            const allChecked = $('.row-checkbox').length === $('.row-checkbox:checked').length;
            $('#checkAllBoxes').prop('checked', allChecked);
            toggleNotifyButton();
        });

        function toggleNotifyButton() {
            const anyChecked = $('.row-checkbox:checked').length > 0;
            $('#notifyButton').prop('disabled', !anyChecked);
        }
    });
</script>
