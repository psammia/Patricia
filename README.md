[HttpPost]
public ActionResult ImportOldBoxesArchive(IFormFile excelFile)
{
    string correlationId = Guid.NewGuid().ToString();

    try
    {
        // Configure validation for OldBoxes Excel file
        var validationConfig = new ExcelValidationConfig
        {
            WorksheetIndex = 1,
            HeaderRow = 1,
            DataStartRow = 2,
            CaseSensitiveHeaders = false,
            MinimumDataRows = 1,
            ExpectedHeaders = new List<ColumnConfig>
            {
                new ColumnConfig
                {
                    Name = "Code",
                    DataType = CellDataType.Text,
                    IsRequired = true,
                    MaxLength = 50
                },
                new ColumnConfig
                {
                    Name = "CompanyName",
                    DataType = CellDataType.Text,
                    IsRequired = true,
                    MaxLength = 200
                },
                new ColumnConfig
                {
                    Name = "IsActive",
                    DataType = CellDataType.Number,
                    IsRequired = true,
                    MinValue = 0,
                    MaxValue = 1,
                    MaxDecimalPlaces = 0
                },
                new ColumnConfig
                {
                    Name = "BoxRef",
                    DataType = CellDataType.Text,
                    IsRequired = true,
                    MustBeUnique = true,
                    MaxLength = 100
                },
                new ColumnConfig
                {
                    Name = "FileName",
                    DataType = CellDataType.Text,
                    IsRequired = true,
                    MaxLength = 255
                },
                new ColumnConfig
                {
                    Name = "FileAdditionalInfo",
                    DataType = CellDataType.Text,
                    IsRequired = false,
                    MaxLength = 500
                },
                new ColumnConfig
                {
                    Name = "BoxStatus",
                    DataType = CellDataType.Text,
                    IsRequired = true,
                    MaxLength = 50
                },
                new ColumnConfig
                {
                    Name = "ArchivingPeriod",
                    DataType = CellDataType.Number,
                    IsRequired = true,
                    MinValue = 0,
                    MaxValue = 99,
                    MaxDecimalPlaces = 0
                },
                new ColumnConfig
                {
                    Name = "BoxSentBy",
                    DataType = CellDataType.Text,
                    IsRequired = true,
                    MaxLength = 100
                },
                new ColumnConfig
                {
                    Name = "BoxSentDate",
                    DataType = CellDataType.Date,
                    IsRequired = true,
                    DisallowFutureDates = true
                },
                new ColumnConfig
                {
                    Name = "LastIndex",
                    DataType = CellDataType.Number,
                    IsRequired = true,
                    MinValue = 0,
                    MaxDecimalPlaces = 0
                }
            }
        };

        // Validate the Excel file
        var validationService = new ExcelValidationService();
        var validationResult = validationService.ValidateExcelFile(excelFile, validationConfig);

        if (!validationResult.IsValid)
        {
            var errorMessages = string.Join("\n", validationResult.Errors.Select(e => $"{e.Location}: {e.Message}"));
            return Json(new 
            { 
                isSuccess = false, 
                message = $"File validation failed:\n{errorMessages}", 
                correlationId = correlationId 
            });
        }

        // If validation passes, process the file
        ImportOldBoxesReq req = new ImportOldBoxesReq()
        {
            BaseReq = new BaseRequest(
                HttpContext,
                GetSession("ArchiveData")),
            OldBoxesList = GetOldBoxes(excelFile!)
        };

        correlationId = req.BaseReq.CorrelationId;

        ImportOldBoxesRes resp = Common.ApiCall<ImportOldBoxesRes>(req, "ImportOldBoxes");

        if (resp.WebResp.HttpResponseCode != HttpStatusCode.OK)
        {
            return Json(new
            {
                isSuccess = false,
                message = resp.WebResp.ResponseMessage,
                correlationId = req.BaseReq.CorrelationId
            });
        }

        return Json(new
        {
            isSuccess = true,
            message = "File has been successfully processed!"
        });
    }
    catch (Exception ex)
    {
        Dictionary<string, object> dic = Common.GenerateVariables(HttpContext, GetSession("ArchiveData")!);

        LogError(ex.Message, new CorrelationInfo()
        {
            CorrelationId = correlationId,
            UserName = dic["user"].ToString(),
            RequestURL = "File/Import",
            StatusCode = HttpStatusCode.InternalServerError,
            RDirection = RequestDirection.Response,
            RType = RequestType.POST,
        }, ex);

        return Json(new
        {
            isSuccess = false,
            message = $"{ex.ToString()}",
            correlationId = correlationId
        });
    }
}

private List<OldBox> GetOldBoxes(IFormFile file)
{
    List<OldBox> result = new List<OldBox>();

    using (Stream stream = file.OpenReadStream())
    {
        using (XLWorkbook workbook = new XLWorkbook(stream))
        {
            IXLWorksheet workSheet = workbook.Worksheets.First();

            Dictionary<string, int> headers = workSheet.Row(1).Cells()
                .Where(c => !string.IsNullOrWhiteSpace(c.GetString()))
                .ToDictionary(c => c.GetString().Trim(), c => c.Address.ColumnNumber);

            foreach (IXLRow row in workSheet.RowsUsed().Skip(1))
            {
                OldBox record = new OldBox
                {
                    Code = row.Cell(headers["Code"]).GetString().Trim(),
                    CompanyName = row.Cell(headers["CompanyName"]).GetString().Trim(),
                    IsActive = int.Parse(row.Cell(headers["IsActive"]).GetString().Trim()),
                    BoxRef = row.Cell(headers["BoxRef"]).GetString().Trim(),
                    FileName = row.Cell(headers["FileName"]).GetString().Trim(),
                    FileAdditionalInfo = row.Cell(headers["FileAdditionalInfo"]).GetString().Trim(),
                    BoxStatus = row.Cell(headers["BoxStatus"]).GetString().Trim(),
                    ArchivingPeriod = int.Parse(row.Cell(headers["ArchivingPeriod"]).GetString().Trim()),
                    BoxSentBy = row.Cell(headers["BoxSentBy"]).GetString().Trim(),
                    BoxSentDate = row.Cell(headers["BoxSentDate"]).GetString().Trim(),
                    LastIndex = long.Parse(row.Cell(headers["LastIndex"]).GetString().Trim())
                };

                result.Add(record);
            }
        }
    }

    return result;
}

// OLD ValidateFiles METHOD IS NOW REPLACED BY ExcelValidationService
// You can remove this method entirely
