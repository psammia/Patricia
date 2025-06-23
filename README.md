<script>
    $(document).ready(function () {
        let table = $("#TblContainertoNotifyWarehouseTable").DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        // Enable/Disable Notify button
        function toggleNotifyButton() {
            const selectedCount = $('.row-checkbox:checked').length;
            $('#notifyButton').prop('disabled', selectedCount === 0);
        }

        // Check all boxes
        $(document).on('change', '#checkAllBoxes', function () {
            const isChecked = $(this).is(':checked');
            $('.row-checkbox').prop('checked', isChecked);
            toggleNotifyButton();
        });

        // Individual checkbox change
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

            console.log("Sending data to server:", data);

            $.ajax({
                type: 'POST',
                url: '/BoxEntity/NotifyWareHouse/',
                data: data,
                success: function (notifyResponse) {
                    if (notifyResponse) {
                        // After success, reload the updated table content from server
                        $.ajax({
                            type: 'POST',
                            url: '/BoxEntity/GetContainerToNotifyWarehouse/',
                            data: {},
                            dataType: 'html',
                            success: function (htmlResponse) {
                                // Destroy the existing table before replacing its content
                                table.destroy();

                                // Replace tbody with updated content
                                const newTbody = $(htmlResponse).find('tbody').html();
                                $('#TblContainertoNotifyWarehouseTable tbody').html(newTbody);

                                // Re-initialize DataTable
                                table = $('#TblContainertoNotifyWarehouseTable').DataTable({
                                    pagingType: 'full_numbers',
                                    scrollX: true
                                });

                                // Reset checkboxes and buttons
                                $('#checkAllBoxes').prop('checked', false);
                                toggleNotifyButton();
                            },
                            error: function (xhr) {
                                $('#MainRenderLocation').html(xhr.responseText);
                            }
                        });
                    }
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                }
            });
        });
    });
</script>
