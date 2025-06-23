success: function (response) {
    if (response) {
        // Reload the table body using the fresh HTML partial from the server
        $('#TblContainertoNotifyWarehouseTable').DataTable().destroy(); // Destroy old DataTable
        $('#TblContainertoNotifyWarehouseTable').html($(response).find('tbody').html()); // Replace tbody only
        const table = $('#TblContainertoNotifyWarehouseTable').DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        toggleNotifyButton();
        $('#checkAllBoxes').prop('checked', false);
    }
}
