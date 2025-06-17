$(document).ready(function () {
    var table = $('#myDataTable').DataTable();

    $('#btnSearch').on('click', function () {
        // Your custom logic to reload or filter the DataTable
        table.ajax.reload(null, false); // if using Ajax

        // Use `draw()` event to detect when the table is updated
        $('#myDataTable').on('draw.dt', function () {
            var dataCount = table.rows({ filter: 'applied' }).data().length;

            if (dataCount > 0) {
                $('#btnGenerateReport').prop('disabled', false);
            } else {
                $('#btnGenerateReport').prop('disabled', true);
            }
        });
    });
});
