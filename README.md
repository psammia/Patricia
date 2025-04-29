<script>
    function CancelInvoice(invoiceRef) {
        swal({
            title: "",
            text: "Do you really want to delete this invoice?",
            icon: "warning",
            buttons: true,
            dangerMode: true,
        }).then((willDelete) => {
            if (willDelete) {
                $.ajax({
                    url: '/Invoice/CancelPendingInvoiceFromBranch',
                    type: 'POST',
                    data: {
                        invoiceRef: invoiceRef
                    },
                    success: function () {
                        swal("Deleted!", "Record deleted.", "success").then(() => {
                            location.reload();
                        });
                    },
                    error: function () {
                        swal("Error!", "There was a problem deleting the record.", "error");
                    }
                });
            }
        });
    }
</script>




swal("Error", "you have to fill ft value", "error");
sweetalert.min.js


I want to add are you sure with yes no and I want to track the function on yes
