@using Alterna.Archive.Core.Models
@model Alterna.Archive.Core.Models.TableModel.ContainerToNotifyWarehouseTableModel

<table id="TblContainertoNotifyWarehouseTable" class="table table-striped table-bordered" style="width:100%;">
    <thead>
        <tr>
            <th></th>
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
                        <i id="@iconId" class="fa-regular fa-pen-to-square icon-edit" title="Edit Box Details" style="cursor: pointer;" onclick="onEdit('@container.Id', '@container.Code')"></i>
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
