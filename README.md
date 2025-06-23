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

            console.log('Sending data:', data);

            $.ajax({
                type: 'POST',
                url: '/BoxEntity/NotifyWareHouse/',
                data: data,
                success: function (response) {
                    if (response) {
                        // Properly remove checked rows from DataTable
                        table.rows().every(function () {
                            const row = this.node();
                            const checkbox = $(row).find('.row-checkbox');
                            if (checkbox.is(':checked')) {
                                this.remove(); // Remove from DataTables
                            }
                        });

                        table.draw(false); // Redraw table (no page reset)
                        toggleNotifyButton();
                        $('#checkAllBoxes').prop('checked', false);
                    }

                    console.log('Notify completed');
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                }
            });
        });
    });
</script>
