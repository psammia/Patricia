                var extension = Path.GetExtension(file.FileName)?.ToLowerInvariant();
                if ((extension != ".xlsx") || (extension != ".xlsm"))
                {
                    result.AddError("File", $"Invalid file extension. Expected .xlsx but got {extension}");
                    return false;
                } 

                What s wrong with this syntax, i got this error, while i uploaded a .xlsx file?

                
Error! File validation failed: File: Invalid file extension. Expected .xlsx but got .xlsx
Correlation Id: 7e4a4c9c-1d5a-427c-8e4b-7f142b694fc2


public class YourController : Controller
{
    private readonly ExcelValidationService _validationService;
    
    // Constructor - inject the service
    public YourController(ExcelValidationService validationService)
    {
        _validationService = validationService;
    }
    
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

            // Use the injected service
            var validationResult = _validationService.ValidateExcelFile(excelFile, validationConfig);

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
    
    // Keep your existing GetOldBoxes method as is
    private List<OldBox> GetOldBoxes(IFormFile file)
    {
        // ... existing code
    }
}
