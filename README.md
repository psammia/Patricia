@{
    ViewBag.Title = "Notify Warehouse";
}

<div class="row">
    <div class="col-md-12">
        <h3>Notify Warehouse</h3>
    </div>
    <div class="col-md-12">
        <ol class="breadcrumb sgbl-breadcrumb">
            <li><a href="~/Home/Index/Redirect">Home</a></li>
            <li class="active">List of Boxes to Sent to Warehouse</li>
        </ol>
    </div>
</div>

<div class="card">
    <div class="card-header"></div>
    <div class="card-content collapse show">
        <div class="card-body card-dashboard">
            <div id="TableDisplay" class="table-spacer">
            </div>
            <br />
        </div>
    </div>
</div>

<script>
    $(document).ready(() => {
        $.ajax({

            type: 'POST',
            url: '/BoxRCA/GetContainerToNotifyWarehouse/',
            data: {
            },
            dataType: 'html',
            success: function (response) {
                $('#TableDisplay').html(response);
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
        return false;
    });
</script>
