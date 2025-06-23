<script>
    $(document).ready(() => {
        const table = $("#TblContainertoNotifyWarehouseTable").DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        // Enable/Disable Notify button
        function toggleNotifyButton() {
            const selectedCount = $('.row-checkbox:checked').length;
            $('#notifyButton').prop('disabled', selectedCount === 0);
        }

        // Check all functionality
        $(document).on('change', '#checkAllBoxes', function () {
            const isChecked = $(this).is(':checked');
            $('.row-checkbox').prop('checked', isChecked);
            toggleNotifyButton();
        });

        // Individual checkbox toggle
        $(document).on('change', '.row-checkbox', function () {
            const allChecked = $('.row-checkbox').length === $('.row-checkbox:checked').length;
            $('#checkAllBoxes').prop('checked', allChecked);
            toggleNotifyButton();
        });

        // Notify button click
        $('#notifyButton').on('click', function () {
            const selectedIds = $('.row-checkbox:checked').map(function () {
                return $(this).val();
            }).get();

            const data = {
                containerIds: selectedIds.join(',')
            };

            console.log(data);

            $.ajax({
                type: 'POST',
                url: '/BoxEntity/NotifyWareHouse/',
                data: data,
                success: function (response) {
                    if (response) {
                        $('.row-checkbox:checked').each(function () {
                            const row = $(this).closest('tr');
                            table.row(row).remove(); // Remove from DataTable
                        });

                        table.draw(false); // Redraw without resetting pagination
                        toggleNotifyButton();

                        // Uncheck the "check all" box
                        $('#checkAllBoxes').prop('checked', false);
                    }

                    console.log('Notify complete');
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                }
            });
        });
    });
</script>
