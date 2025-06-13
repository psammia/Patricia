✅ Adjusted NotifyWarehouse.cshtml (Main View)
cs
Copy
Edit
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>

<script>
    let table;

    $(document).ready(() => {
        loadTable();

        $('#notifyButton').on('click', function () {
            const selectedIds = $('.row-checkbox:checked').map(function () {
                return $(this).val();
            }).get();

            if (selectedIds.length === 0) return;

            Swal.fire({
                title: 'Confirm Notification',
                text: `Notify warehouse for ${selectedIds.length} box(es)?`,
                icon: 'question',
                showCancelButton: true,
                confirmButtonText: 'Yes, notify',
                cancelButtonText: 'Cancel'
            }).then(result => {
                if (result.isConfirmed) {
                    $('#notifyButton').prop('disabled', true); // optional UX
                    $.post('/BoxRCA/NotifyWareHouse/', { containerIds: selectedIds.join(',') })
                        .done(() => {
                            Swal.fire('Success', 'Warehouse has been notified.', 'success');
                            loadTable();
                        })
                        .fail(xhr => {
                            Swal.fire('Error', 'Something went wrong.', 'error');
                            $('#MainRenderLocation').html(xhr.responseText);
                        });
                }
            });
        });
    });

    function loadTable() {
        $.post('/BoxRCA/GetContainerToNotifyWarehouse/', {}, function (html) {
            $('#TableDisplay').html(html);
            bindCheckboxHandlers();
            initDataTable();
        });
    }

    function initDataTable() {
        if ($.fn.DataTable.isDataTable('#TblContainertoNotifyWarehouseTable')) {
            $('#TblContainertoNotifyWarehouseTable').DataTable().destroy();
        }
        table = $('#TblContainertoNotifyWarehouseTable').DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });
    }

    function bindCheckboxHandlers() {
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

        toggleNotifyButton();
    }
</script>
✅ Your Partial View Should Only Contain the Table
Remove all <script> tags from the partial view.

cshtml
Copy
Edit
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
