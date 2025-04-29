VM27:3 Uncaught ReferenceError: Swal is not defined
    at CancelInvoice (<anonymous>:3:9)
    at SVGSVGElement.onclick (PendingInvoices:1:1)



PendingInvoices:1 Uncaught ReferenceError: B2017 is not defined
    at SVGSVGElement.onclick (PendingInvoices:1:15)





<script>
    function CancelInvoice(invoiceRef) {
        Swal.fire({
            title: '',
            text: "Do you really want to delete this invoice?",
            icon: 'warning',
            showCancelButton: true,
            confirmButtonColor: '#d33',
            cancelButtonColor: '#3085d6',
            confirmButtonText: 'Yes'
        }).then((result) => {
            if (result.isConfirmed) {
                $.ajax({
                    url: '/Invoice/CancelPendingInvoiceFromBranch',
                    type: 'POST',
                    data: {
                        invoiceRef: invoiceRef
                    },
                    success: function () {
                        Swal.fire('Deleted!', 'Record deleted.', 'success').then(() => {
                            location.reload();
                        });
                    },
                    error: function () {
                        Swal.fire('Error!', 'There was a problem deleting the record.', 'error');
                    }
                });
            }
        });
    }
</script>





@model Alterna_Port_Frontend.Models.CustomModels.PendingInvoicesByChannelModel
@using Alterna_Port_Frontend.Models

@if (Model.PendingInvoicesList != null && Model.PendingInvoicesList.Any())
{
    <table class="table table-bordered table-striped" style="width:100%" id="table-pending-invoices">
        <thead>
            <tr>
                <th></th>
                <th>Invoice Ref</th>
                <th>Client No</th>
                <th>Client Name</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var invoice in Model.PendingInvoicesList)
            {
                <tr>
                    <td class="td-icons-wrapper">
                        @*                         <a href="/Invoice/CancelPendingInvoiceFromBranch?InvoiceRef=@invoice.InvoiceRef" class="no-underline"> *@
                        <i class="fa-solid fa-xmark fa text-danger td-icon pr-3" onClick="CancelInvoice(@invoice.InvoiceRef)" aria-hidden="true" title="Cancel Transaction"></i>
                        @* /a> *@
                        <a href="/Invoice/Search?InvoiceRef=@invoice.InvoiceRef" class="no-underline">
                            <i class="fa-solid fa-file-lines fa text-info td-icon" aria-hidden="true" title="View Invoice"></i>
                        </a>
                    </td>
                    <td>@invoice.InvoiceRef </td>
                    <td>@invoice.ClientNumber</td>
                    <td>@invoice.ClientName</td>
                </tr>
            }
        </tbody>
    </table>
}
else
{
    <div class="alert alert-danger" role="alert">
        <div class="">
            <span>No pending invoices found for the selected channel.</span>
        </div>
    </div>
}

<script>
    function CancelInvoice(invoiceRef) {
        swal.fire({
            title: '',
            text: "Do you really want to delete this invoice?",
            icon: 'warning',
            showCancelButton: true,
            confirmButtonColor:'#d33',
            cancelButtonColor:'#3085d6',
            confirmButtonText: 'Yes'
        }).then((result) => {
            if (result.isConfirmed) {
                $ajax.({
                    url: '/Invoice/CancelPendingInvoiceFromBranch?InvoiceRef=invoiceRef',
                    type:'POST',
                        data: {
                        invoiceRef: invoiceRef
                    },
                    success: function (invoiceRef) {
                        swal.fire('Deleted!', 'Record deleted.', 'succes').then(() => {
                            location.reload();
                        });
                    }, error: function () { swal.fire('Error!', 'There was a problem deleting the record.', 'error'); }
                });
            }
        });
    }

</script>
