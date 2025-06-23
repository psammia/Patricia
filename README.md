<script>
    let table;

    $(document).ready(() => {
        getContainerToNotifyWarehouseData();

        // Bind click event for notify button here
        $('#notifyButton').on('click', function () {
            const selectedIds = $('.row-checkbox:checked').map(function () {
                return $(this).val();
            }).get();

            if (selectedIds.length === 0) return;

            const data = {
                containerIds: selectedIds.join(',')
            };

            $.ajax({
                type: 'POST',
                url: '/BoxEntity/NotifyWareHouse/',
                data: data,
                success: function (notifyResponse) {
                    if (notifyResponse) {
                        getContainerToNotifyWarehouseData(); // Reload table after notify
                    }
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                }
            });
        });
    });

    function getContainerToNotifyWarehouseData() {
        $.ajax({
            type: 'POST',
            url: '/BoxEntity/GetContainerToNotifyWarehouse/',
            dataType: 'html',
            success: function (response) {
                $('#TableDisplay').html(response);

                // Re-initialize the DataTable
                if ($.fn.DataTable.isDataTable('#TblContainertoNotifyWarehouseTable')) {
                    $('#TblContainertoNotifyWarehouseTable').DataTable().destroy();
                }

                table = $('#TblContainertoNotifyWarehouseTable').DataTable({
                    pagingType: 'full_numbers',
                    scrollX: true
                });

                bindCheckboxLogic(); // Rebind checkbox logic every time table is reloaded
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
    }

    function bindCheckboxLogic() {
        // Enable/disable Notify button based on checkbox selection
        function toggleNotifyButton() {
            const selectedCount = $('.row-checkbox:checked').length;
            $('#notifyButton').prop('disabled', selectedCount === 0);
        }

        // Handle Check All
        $(document).off('change', '#checkAllBoxes').on('change', '#checkAllBoxes', function () {
            const isChecked = $(this).is(':checked');
            $('.row-checkbox').prop('checked', isChecked);
            toggleNotifyButton();
        });

        // Handle individual checkbox
        $(document).off('change', '.row-checkbox').on('change', '.row-checkbox', function () {
            const allChecked = $('.row-checkbox').length === $('.row-checkbox:checked').length;
            $('#checkAllBoxes').prop('checked', allChecked);
            toggleNotifyButton();
        });

        // Initial state check
        toggleNotifyButton();
    }
</script>
