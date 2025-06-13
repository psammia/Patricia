 Final JavaScript for #notifyButton:
js
Copy
Edit
$(document).on('click', '#notifyButton', function () {
    const selectedCheckboxes = $('.row-checkbox:checked');

    const selectedIds = selectedCheckboxes.map(function () {
        return $(this).val();
    }).get();

    if (selectedIds.length === 0) return;

    Swal.fire({
        title: 'Are you sure?',
        text: `You are about to notify ${selectedIds.length} box(es) to the warehouse.`,
        icon: 'warning',
        showCancelButton: true,
        confirmButtonText: 'Yes, notify',
        cancelButtonText: 'Cancel',
        reverseButtons: true
    }).then((result) => {
        if (result.isConfirmed) {
            $('#loadingOverlay').show(); // Show spinner

            $.ajax({
                type: 'POST',
                url: '/BoxRCA/NotifyWareHouse/',
                data: { containerIds: selectedIds.join(',') },
                success: function () {
                    // Re-fetch the table data from the server
                    getContainerToNotifyWarehouseData();

                    Swal.fire({
                        icon: 'success',
                        title: 'Notified!',
                        text: 'Selected boxes were successfully notified.'
                    });
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                    Swal.fire('Error', 'Something went wrong while notifying the warehouse.', 'error');
                },
                complete: function () {
                    $('#loadingOverlay').hide(); // Hide spinner
                }
            });
        }
    });
});
🔁 Update Your getContainerToNotifyWarehouseData to Rebind Table
Make sure after reload, the table is reinitialized:

js
Copy
Edit
function getContainerToNotifyWarehouseData() {
    $.ajax({
        type: 'POST',
        url: '/BoxRCA/GetContainerToNotifyWarehouse/',
        data: {},
        dataType: 'html',
        success: function (response) {
            $('#TableDisplay').html(response);
            initNotifyWarehouseTable(); // Re-init your DataTable and checkbox bindings
        },
        error: function (xhr) {
            $('#MainRenderLocation').html(xhr.responseText);
        }
    });
}
