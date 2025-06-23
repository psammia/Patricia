success: function (response) {
    if (response) {
        // Loop through each checked checkbox
        $('.row-checkbox:checked').each(function () {
            const tr = $(this).closest('tr');    // Get the row DOM element
            const row = table.row(tr);           // Get the DataTables row
            row.remove();                        // Remove it from DataTables
        });

        toggleNotifyButton();
        table.draw(false); // false = don't reset paging
    }

    console.log('one time ');
}
