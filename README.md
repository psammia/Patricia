@using Alterna.Archive.Core.Models.SearchModel
@model Alterna.Archive.Core.Models.SearchModel.WarehouseContainerSearchModel
@{
    ViewBag.Title = "Warehouse Search Boxes";
}

<div class="row">
    <div class="col-md-12">
        <h3>Warehouse Management</h3>
    </div>
    <div class="col-md-12">
        <ol class="breadcrumb sgbl-breadcrumb">
            <li><a href="~/Home/Index/Redirect">Home</a></li>
            <li class="active">Search Box</li>
        </ol>
    </div>
</div>

<div class="card">
    <div class="card-header"></div>
    <div class="card-content collapse show">
        <div class="card-body card-dashboard">
            <div class="row" id="WarehouseContainersFilterOptions">

                @{
                    var currentFormattedDate = "Today " + @DateTime.Now.ToString("dd MMMM, yyyy");
                }

                <div class="form-group col-md-4">
                    <label>From Date </label>
                    <input id="dpFromDate" class="datepicker form-control" placeholder="Select A Date" />
                </div>

                <div class="form-group col-md-4">
                    <label>To Date </label>
                    <input id="dpToDate" class="datepicker form-control" placeholder="Select A Date" />
                </div>

                <div class="form-group col-md-4">
                    <label>Box Ref </label>
                    <input id="IContainerCode" type="text" class="form-control">
                </div>

                <div class="form-group col-md-4">
                    <label>Branch </label>
                    <select class="form-control selectpicker" id="SCompanyCode" data-live-search="true">
                        <option value="">No Branch Selected</option>
                        @{
                            foreach ((String companyCode, String companyName) in Model.CompaniesDict)
                            {
                                <option value="@companyCode">@companyCode - @companyName</option>
                            }
                        }
                    </select>
                </div>

                <div class="form-group col-md-4">
                    <label>Box Status</label>
                    <select class="form-control selectpicker" id="StatusCode" data-live-search="true">
                        <option value="">All</option>
                        <option value="SENT">SENT</option>
                        <option value="RECEIVED">RECEIVED</option>
                        <option value="TOBEDESTR">TOBEDESTR</option>
                        <option value="DESTROYED">DESTROYED</option>
                    </select>
                </div>
                @*
                <div class="form-group col-md-4">
                </div> *@@*
                <div class="form-group col-md-4">
                </div> *@
                
                <div class="col-md-4">
                    <div>&#8291;</div>
                    <button type="button" class="btn btn-primary" style="margin-top: 6px" onclick="showData()">Search</button>
                    <button type="button" class="btn btn-success" style="margin-top: 6px" onclick="GenerateReport()">Generate Report</button>
                </div>


            </div>

            <div id="TableDisplay" class="table-spacer"></div>
        </div>
    </div>
</div>

<script>
    $('.datepicker').pickadate({
        labelMonthNext: 'Go to the next month',
        labelMonthPrev: 'Go to the previous month',
        labelMonthSelect: 'Pick a month from the dropdown',
        labelYearSelect: 'Pick a year from the dropdown',
        selectMonths: true,
        selectYears: 20,
        closeOnClear: false,
        firstDay: 1
    });

    function showData() {

        $.ajax({
            type: 'POST',
            url: '/Warehouse/GetWarehouseContainer/',
            data: {
                searchModel:
                {
                    CompanyCode: $('#SCompanyCode').find(":selected").val(),
                    Code: $('#IContainerCode').val(),
                    FromDate: $('#dpFromDate').val(),
                    ToDate: $('#dpToDate').val(),
                    StatusCode: $('#StatusCode').val()
                }
            },
            dataType: 'html',
            success: function (response) {
                if (!ValidationBetweenDates()) {
                    return;
                }
                $("#WarehouseContainersFilterOptions").hide();
                $('#TableDisplay').html(response);
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
        return false;
    }


    function GenerateReport() {

        $.ajax({
            type: 'POST',
            url: '/Warehouse/GetWarehouseContainer/',
            data: {
                searchModel:
                {
                    CompanyCode: $('#SCompanyCode').find(":selected").val(),
                    Code: $('#IContainerCode').val(),
                    FromDate: $('#dpFromDate').val(),
                    ToDate: $('#dpToDate').val(),
                    StatusCode: $('#StatusCode').val()
                }
            },
            dataType: 'html',
            success: function (response) {
                if (!ValidationBetweenDates()) {
                    return;
                }
                $("#WarehouseContainersFilterOptions").hide();
                $('#TableDisplay').html(response);
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
        return false;
    }

    function ValidationBetweenDates() {
        var from = $("#dpFromDate").val();
        var to = $("#dpToDate").val();

        if (Date.parse(from) > Date.parse(to)) {
            swal("Invalid Date Range", "End Date must be greater than Start Date", "error");
            return false;
        }
        else {
            return true;
        }

        function downloadPDF() {
            window.open('@Url.Action("DownloadPDF", "Configuration")?boxReference=' + $('#boxReference').val(), '_blank').focus();
        }

    }
</script>
