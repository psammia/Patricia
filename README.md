$('#notifyButton').on('click', function () {
    const selectedIds = $('.row-checkbox:checked').map(function () {
        return $(this).val();
    }).get();



    const data = 
    {
        containerIds: selectedIds.join(',')
    }
    console.log(data)

    table.rows().every(function (){
        const row = this.node();

        const isChecked = $('.row-checkbox:checked').closest('#'+ row.id );
        console.log(isChecked[0])
        
        if (isChecked[0]) isChecked[0].remove()

        setTimeout(() => {
            table.draw()



        }, 1000);

    })
