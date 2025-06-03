
main view
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
            <button id="notifyButton" class="btn btn-primary" disabled>Notify Warehouse</button>
        </div>
        <div id="TableDisplay" class="table-spacer">
        </div>
        <br />
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

partial view
@using Alterna.Archive.Core.Models
@model Alterna.Archive.Core.Models.TableModel.ContainerToNotifyWarehouseTableModel

<table id="TblContainertoNotifyWarehouseTable" class="table table-striped table-bordered" style="width:100%;">
    <thead>
        <tr>
            <th >
                <input type="checkbox" id="checkAllBoxes" />
            </th>
            <th>Box Ref</th>
            <th>Box Type</th>
            <th>Status Code</th>
            <th>Last Modified By</th>
            <th>Last Modified Date</th>
        </tr>
    </thead>
    <tbody>
        @if (Model.ContainersToNotifyWarehouseList.Count > 0)
        {
            foreach (Container container in Model.ContainersToNotifyWarehouseList)
            {
                var iconId = container.Id;
                var trId = "Row" + container.Id;

                <tr id="@trId">
                    <td class="text-center">
                        <input type="checkbox" class="row-checkbox" value="@container.Id" />
                    </td>
                    <td>@container.Code</td>
                    <td>@container.ContainerType</td>
                    <td>@container.StatusCode</td>
                    <td>@container.LastModifiedBy</td>
                    <td>@container.LastModifiedDate.ToString("dd/MM/yyyy")</td>
                </tr>

            }
        }
    </tbody>
</table>

<script>
    $(document).ready(() => {
        const table = $("#TblContainertoNotifyWarehouseTable").DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        // Check all functionality
        $('#checkAllBoxes').on('change', function () {
            $('.row-checkbox').prop('checked', this.checked);
            toggleNotifyButton();
        });

        // Individual checkbox toggle
        $(document).on('change', '.row-checkbox', function () {
            const allChecked = $('.row-checkbox').length === $('.row-checkbox:checked').length;
            $('#checkAllBoxes').prop('checked', allChecked);
            toggleNotifyButton();
        });

        // Enable/Disable Notify button
        function toggleNotifyButton() {
            const selectedCount = $('.row-checkbox:checked').length;
            $('#notifyButton').prop('disabled', selectedCount === 0);
        }
        $('#notifyButton').on('click', function () {
            const selectedIds = $('.row-checkbox:checked').map(function () {
                return $(this).val();
            }).get();

            console.log("Selected IDs:", selectedIds);
            // Submit via AJAX or redirect as needed
        });
    });
</script>






