$(document).ready(function () {
  const table = $('#myTable').DataTable(); // initialize DataTable

  $('#deleteRows').on('click', function () {
    // Loop through all rows (including non-visible ones, if needed)
    table.rows().every(function () {
      const row = this.node(); // get the actual DOM node of the row
      const checkbox = $(row).find('.row-check');

      if (checkbox.is(':checked')) {
        this.remove(); // use DataTables API to remove the row
      }
    });

    table.draw(); // redraw the table to update the UI
  });
});