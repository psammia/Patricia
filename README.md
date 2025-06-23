This is my main view : NotifyWarehouse.cshtml


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
            <li class="active">List of Boxes to be Sent to Warehouse</li>
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
        getContainerToNotifyWarehouseData();
    });

    function getContainerToNotifyWarehouseData() {
        $.ajax({
            type: 'POST',
            url: '/BoxEntity/GetContainerToNotifyWarehouse/',
            data: {},
            dataType: 'html',
            success: function (response) {
                $('#TableDisplay').html(response);
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
    }
</script>

and the code fot the partial view: _GetContainerToNotifyWarehouse.cshtml
@using Alterna.Archive.Core.Models
@model Alterna.Archive.Core.Models.TableModel.ContainerToNotifyWarehouseTableModel

<table id="TblContainertoNotifyWarehouseTable" class="table table-striped table-bordered" style="width:100%;">
    <thead>
        <tr>
            <th class="text-center align-middle" style="padding: 6px; min-width: 66px">
                <div class="text-center">
                    <input type="checkbox" id="checkAllBoxes" class="form-check-input" />
                </div>
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
                    <td class="text-center align-middle">
                        <div class="text-center">
                            <input type="checkbox" class="form-check-input row-checkbox" value="@container.Id" />
                        </div>
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
    $(document).ready(function () {
        let table = $("#TblContainertoNotifyWarehouseTable").DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        // Enable/Disable Notify button
        function toggleNotifyButton() {
            const selectedCount = $('.row-checkbox:checked').length;
            $('#notifyButton').prop('disabled', selectedCount === 0);
        }

        // Check all boxes
        $(document).on('change', '#checkAllBoxes', function () {
            const isChecked = $(this).is(':checked');
            $('.row-checkbox').prop('checked', isChecked);
            toggleNotifyButton();
        });

        // Individual checkbox change
        $(document).on('change', '.row-checkbox', function () {
            const allChecked = $('.row-checkbox').length === $('.row-checkbox:checked').length;
            $('#checkAllBoxes').prop('checked', allChecked);
            toggleNotifyButton();
        });

        // Notify button click
        $('#notifyButton').on('click', function () {
            const selectedIds = $('.row-checkbox:checked').map(function () {
                return $(this).val();
            }).get();

            const data = {
                containerIds: selectedIds.join(',')
            };

            console.log("Sending data to server:", data);

            $.ajax({
                type: 'POST',
                url: '/BoxEntity/NotifyWareHouse/',
                data: data,
                success: function (notifyResponse) {
                    if (notifyResponse) {
                        // After success, reload the updated table content from server
                        $.ajax({
                            type: 'POST',
                            url: '/BoxEntity/GetContainerToNotifyWarehouse/',
                            data: {},
                            dataType: 'html',
                            success: function (htmlResponse) {
                                // Destroy the existing table before replacing its content
                                table.destroy();

                                // Replace tbody with updated content
                                const newTbody = $(htmlResponse).find('tbody').html();
                                $('#TblContainertoNotifyWarehouseTable tbody').html(newTbody);

                                // Re-initialize DataTable
                                table = $('#TblContainertoNotifyWarehouseTable').DataTable({
                                    pagingType: 'full_numbers',
                                    scrollX: true
                                });

                                // Reset checkboxes and buttons
                                $('#checkAllBoxes').prop('checked', false);
                                toggleNotifyButton();
                            },
                            error: function (xhr) {
                                $('#MainRenderLocation').html(xhr.responseText);
                            }
                        });
                    }
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                }
            });
        });
    });
</script>

the last code provided by you, didnt work. give another solution or code. full code

