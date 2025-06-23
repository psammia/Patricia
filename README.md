$('#notifyButton').on('click', function () {
    const selectedIds = $('.row-checkbox:checked').map(function () {
        return $(this).val();
    }).get();

    const data = {
        containerIds: selectedIds.join(',')
    };
    console.log(data);

    table.rows().every(function () {
        const rowData = this.data(); // get row data

        // Assuming your checkbox value matches something in the rowData
        if (selectedIds.includes(rowData.id.toString())) {
            this.remove(); // This is the correct way to remove the row
        }
    });

    table.draw(false); // false means don't reset the paging
});
