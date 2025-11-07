Archiving BLL.cs
====================
        public void ImportOldBoxes(ImportOldBoxesReq req)
        {
            DAL.DAL _dal = new();

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__Old_Boxes", GetOldBoxesDt(req.OldBoxesList).AsTableValuedParameter());
            parameters.Add("P__User", req.BaseReq.CurrentUser);

            _dal.ExecuteQuery<Container>(
                "usp_Insert_Into_All_Tables",
                parameters,
                CommandType.StoredProcedure,
               CommandDirection.Update);
        }

        public DataTable GetOldBoxesDt(List<OldBoxDto> oldBoxes)
        {
            int rowNbr = 1;

            DataTable dt = new DataTable("TVP_Old_Boxes");

            dt.Columns.Add("RowId");
            dt.Columns.Add("Code");
            dt.Columns.Add("CompanyName");
            dt.Columns.Add("Mnemonic");
            dt.Columns.Add("IsActive");
            dt.Columns.Add("BoxRef");
            dt.Columns.Add("FileName");
            dt.Columns.Add("AdditionalInfo");
            dt.Columns.Add("StatusCode");
            dt.Columns.Add("ArchivingPeriod");
            dt.Columns.Add("BoxSentBy");
            dt.Columns.Add("BoxSentDate");
            dt.Columns.Add("LastIndex");

            oldBoxes.ForEach(bx=>
            {
                DataRow dr = dt.NewRow();
                dr["RowId"] = rowNbr;
                dr["Code"] = bx.Code?.Trim();
                dr["CompanyName"] = bx.CompanyName?.Trim();
                dr["Mnemonic"] = bx.CompanyName?.Trim();
                dr["IsActive"] = bx.IsActive;
                dr["BoxRef"] = bx.BoxRef.Trim();
                dr["FileName"] = bx.FileName.Trim();
                dr["AdditionalInfo"] = bx.FileAdditionalInfo?.Trim();
                dr["StatusCode"] = bx.BoxStatus?.Trim();
                dr["ArchivingPeriod"] = bx.ArchivingPeriod;
                dr["BoxSentBy"] = bx.BoxSentBy?.Trim();
                dr["BoxSentDate"] = bx.BoxSentDate?.Trim();
                dr["LastIndex"] = bx.LastIndex;

                dt.Rows.Add(dr);

                rowNbr++;
            });
            
            return dt;
        }

        =====
        Configuration Controller Front end
        =============
                public Boolean UpdateSequence(Int32 SequenceId, String Owner, String Prefix, Int64 LastIndex, String Suffix, Boolean IsActive)
        {
            UpdateSequenceReq updateSequenceReq = new()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                SequenceId = SequenceId,
                Owner = Owner,
                Prefix = Prefix,
                LastIndex = LastIndex,
                Suffix = Suffix,
                IsActive = IsActive
            };

            UpdateSequenceRes resp = new();

            resp = Common.ApiCall<UpdateSequenceRes>(updateSequenceReq, "UpdateSequence");

            return true;
        }

        public Boolean UpdateConfiguration(Int32 ModelID, String value, Boolean isActive, Int32 index)
        {

            UpdateConfigurationReq updateConfigurationReq = new()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                Id = ModelID,
                SettingValue = value,
                IsActive = isActive,
                SettingName = StaticConfigurationModel.ConfigurationList[index].SettingName,
                SettingDescription = StaticConfigurationModel.ConfigurationList[index].SettingDescription
            };
            UpdateConfigurationRes resp = new();
            resp = Common.ApiCall<UpdateConfigurationRes>(updateConfigurationReq, "UpdateConfiguration");
            return true;

        }

        public ActionResult OldBoxesArchive()
        {
            return View();
        }
        public class ExcelValidationService
        {
            private const long MaxFileSize = 10 * 1024 * 1024; // 10MB
            //private const string ExpectedMimeType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";

            public ValidationResult ValidateExcelFile(IFormFile file, ExcelValidationConfig config)
            {
                var result = new ValidationResult();

                // FILE-LEVEL VALIDATIONS - BASIC FILE CHECKS
                if (!ValidateBasicFileChecks(file, result))
                    return result;

                // FILE-LEVEL VALIDATIONS - SECURITY CHECKS
                //if (!ValidateSecurityChecks(file, result))
                //    return result;

                // STRUCTURE & DATA VALIDATIONS
                try
                {
                    using var stream = file.OpenReadStream();
                    using var workbook = new XLWorkbook(stream);

                    var worksheet = workbook.Worksheet(config.WorksheetIndex);

                    if (worksheet == null)
                    {
                        result.AddError("Worksheet", $"Worksheet at index {config.WorksheetIndex} not found");
                        return result;
                    }

                    // STRUCTURE VALIDATIONS - COLUMN/HEADER VALIDATIONS
                    ValidateHeaders(worksheet, config, result);

                    if (!result.IsValid)
                        return result;

                    // DATA VALIDATIONS
                    ValidateData(worksheet, config, result);

                    // ROW-LEVEL VALIDATIONS
                    ValidateRowCount(worksheet, config, result);
                }
                catch (Exception ex) when (ex.Message.Contains("corrupt") || ex.Message.Contains("invalid") || ex.Message.Contains("not a valid"))
                {
                    result.AddError("File", "File is corrupted and cannot be opened");
                }
                catch (Exception ex)
                {
                    result.AddError("File", $"Error processing file: {ex.Message}");
                }

                return result;
            }

            #region FILE-LEVEL VALIDATIONS - BASIC FILE CHECKS

            private bool ValidateBasicFileChecks(IFormFile file, ValidationResult result)
            {
                // File exists and is not empty (size > 0 bytes)
                if (file == null || file.Length == 0)
                {
                    result.AddError("File", "File is empty or not provided");
                    return false;
                }

                // File size is within acceptable limits (< 10MB)
                if (file.Length > MaxFileSize)
                {
                    result.AddError("File", $"File size exceeds maximum allowed size of {MaxFileSize / (1024 * 1024)}MB");
                    return false;
                }

                // File extension is .xlsx or .xlsm
                var extension = Path.GetExtension(file.FileName)?.ToLowerInvariant();
                if (extension != ".xlsx" && extension != ".xlsm")
                {
                    result.AddError("File", $"Invalid file extension. Expected .xlsx but got {extension}");
                    return false;
                }

                // MIME type validation
                //if (!string.IsNullOrEmpty(file.ContentType) &&
                //    file.ContentType != ExpectedMimeType)
                //{
                //    result.AddError("File", $"Invalid MIME type. Expected {ExpectedMimeType} but got {file.ContentType}");
                //    return false;
                //}

                return true;
            }

            #endregion

            #region FILE-LEVEL VALIDATIONS - SECURITY CHECKS

            //private bool ValidateSecurityChecks(IFormFile file, ValidationResult result)
            //{
            //            // Scan for macros (reject if .xlsm or contains VBA code)
            //            var extension = Path.GetExtension(file.FileName)?.ToLowerInvariant();
            //                //if (extension == ".xlsm")
            //                //{
            //                //    result.AddError("Security", "Macro-enabled files (.xlsm) are not allowed");
            //                //    return false;
            //                //}

            //                // Validate file signature (ZIP signature for xlsx)
            //                using var stream = file.OpenReadStream();
            //    //var signature = new byte[4];
            //    //stream.Read(signature, 0, 4);
            //    //stream.Position = 0;

            //                // Check for ZIP signature: PK (50 4B)
            //                //if (!(signature[0] == 0x50 && signature[1] == 0x4B))
            //                //{
            //                //    result.AddError("Security", "File signature does not match Excel format");
            //                //    return false;
            //                //}

            //    return true;
            //}

            #endregion

            #region STRUCTURE VALIDATIONS - COLUMN/HEADER VALIDATIONS

            private void ValidateHeaders(IXLWorksheet worksheet, ExcelValidationConfig config, ValidationResult result)
            {
                var headerRow = config.HeaderRow;
                var actualHeaders = worksheet.Row(headerRow).Cells()
                    .Where(c => !string.IsNullOrWhiteSpace(c.GetString()))
                    .ToDictionary(c => c.GetString().Trim(), c => c.Address.ColumnNumber);

                foreach (var expectedHeader in config.ExpectedHeaders)
                {
                    // Check if header exists
                    var headerExists = config.CaseSensitiveHeaders
                        ? actualHeaders.ContainsKey(expectedHeader.Name)
                        : actualHeaders.Keys.Any(k => string.Equals(k, expectedHeader.Name, StringComparison.OrdinalIgnoreCase));

                    if (!headerExists)
                    {
                        result.AddError("Headers", $"Missing required column: [{expectedHeader.Name}]");
                    }
                }
            }

            #endregion

            #region DATA VALIDATIONS

            private void ValidateData(IXLWorksheet worksheet, ExcelValidationConfig config, ValidationResult result)
            {
                var startRow = config.DataStartRow;
                var rows = worksheet.RowsUsed().Skip(startRow - 1);

                // Build header dictionary for column lookup
                var headers = worksheet.Row(config.HeaderRow).Cells()
                    .Where(c => !string.IsNullOrWhiteSpace(c.GetString()))
                    .ToDictionary(c => c.GetString().Trim(), c => c.Address.ColumnNumber);

                // Track values for uniqueness validation - BUSINESS LOGIC VALIDATIONS
                var uniquenessTrackers = new Dictionary<string, HashSet<string>>();
                foreach (var header in config.ExpectedHeaders.Where(h => h.MustBeUnique))
                {
                    uniquenessTrackers[header.Name] = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
                }

                int rowNumber = startRow;
                foreach (var row in rows)
                {
                    // Skip completely empty rows
                    if (IsRowEmpty(row))
                    {
                        rowNumber++;
                        continue;
                    }

                    foreach (var header in config.ExpectedHeaders)
                    {
                        if (!headers.ContainsKey(header.Name))
                            continue;

                        var columnIndex = headers[header.Name];
                        var cell = row.Cell(columnIndex);
                        var cellValue = cell.GetString().Trim();

                        ValidateCell(cellValue, header, rowNumber, result, uniquenessTrackers);
                    }

                    rowNumber++;
                }
            }
            private void ValidateCell(
                string cellValue,
                ColumnConfig header,
                int row,
                ValidationResult result,
                Dictionary<string, HashSet<string>> uniquenessTrackers)
            {
                var location = $"Row {row}, Column {header.Name}";

                // CELL-LEVEL CHECKS - Required cells/columns are not empty
                if (header.IsRequired && string.IsNullOrWhiteSpace(cellValue))
                {
                    result.AddError(location, "Required field is empty");
                    return;
                }

                if (string.IsNullOrWhiteSpace(cellValue))
                    return; // Skip further validation for empty optional fields

                // CELL-LEVEL CHECKS - Data types match expectations
                switch (header.DataType)
                {
                    case CellDataType.Text:
                        ValidateText(cellValue, header, location, result);
                        break;

                    case CellDataType.Number:
                        ValidateNumber(cellValue, header, location, result);
                        break;

                    case CellDataType.Date:
                        ValidateDate(cellValue, header, location, result);
                        break;

                    case CellDataType.Boolean:
                        ValidateBoolean(cellValue, location, result);
                        break;

                        //case CellDataType.Email:
                        //    ValidateEmail(cellValue, header, location, result);
                        //    break;

                        //case CellDataType.Phone:
                        //    ValidatePhone(cellValue, location, result);
                        //    break;
                }

                // BUSINESS LOGIC VALIDATIONS - Uniqueness constraints
                if (header.MustBeUnique)
                {
                    if (!uniquenessTrackers[header.Name].Add(cellValue))
                    {
                        result.AddError(location, $"Duplicate value '{cellValue}' found. This field must be unique");
                    }
                }
            }

            // DATA VALIDATION RULE COMPLIANCE - Text length constraints
            private void ValidateText(string value, ColumnConfig config, string location, ValidationResult result)
            {
                if (config.MinLength.HasValue && value.Length < config.MinLength.Value)
                {
                    result.AddError(location, $"Text length {value.Length} is below minimum required length of {config.MinLength.Value}");
                }

                if (config.MaxLength.HasValue && value.Length > config.MaxLength.Value)
                {
                    result.AddError(location, $"Text length {value.Length} exceeds maximum allowed length of {config.MaxLength.Value}");
                }

                // FORMAT VALIDATIONS - Custom regex pattern
                if (!string.IsNullOrEmpty(config.RegexPattern))
                {
                    if (!Regex.IsMatch(value, config.RegexPattern))
                    {
                        result.AddError(location, $"Value '{value}' does not match required format");
                    }
                }
            }

            // CELL-LEVEL CHECKS - Numeric values are within acceptable ranges
            // DATA VALIDATION RULE COMPLIANCE - Numeric constraints (min/max values, decimal places)
            private void ValidateNumber(string value, ColumnConfig config, string location, ValidationResult result)
            {
                if (!decimal.TryParse(value, out decimal numericValue))
                {
                    result.AddError(location, $"Value '{value}' is not a valid number");
                    return;
                }

                if (config.MinValue.HasValue && numericValue < config.MinValue.Value)
                {
                    result.AddError(location, $"Value {numericValue} is below minimum allowed value of {config.MinValue.Value}");
                }

                if (config.MaxValue.HasValue && numericValue > config.MaxValue.Value)
                {
                    result.AddError(location, $"Value {numericValue} exceeds maximum allowed value of {config.MaxValue.Value}");
                }

                // Decimal places validation
                if (config.MaxDecimalPlaces.HasValue)
                {
                    var decimalPlaces = BitConverter.GetBytes(decimal.GetBits(numericValue)[3])[2];
                    if (decimalPlaces > config.MaxDecimalPlaces.Value)
                    {
                        result.AddError(location, $"Value has {decimalPlaces} decimal places but maximum allowed is {config.MaxDecimalPlaces.Value}");
                    }
                }
            }

            // CELL-LEVEL CHECKS - Date formats are valid and parseable
            // DATA VALIDATION RULE COMPLIANCE - Date range validations (not in future, within acceptable range)
            private void ValidateDate(string value, ColumnConfig config, string location, ValidationResult result)
            {
                if (!DateTime.TryParse(value, out DateTime dateValue))
                {
                    result.AddError(location, $"Value '{value}' is not a valid date");
                    return;
                }

                if (config.DisallowFutureDates && dateValue > DateTime.Now)
                {
                    result.AddError(location, $"Future dates are not allowed. Date: {dateValue:yyyy-MM-dd}");
                }

                if (config.MinDate.HasValue && dateValue < config.MinDate.Value)
                {
                    result.AddError(location, $"Date {dateValue:yyyy-MM-dd} is before minimum allowed date of {config.MinDate.Value:yyyy-MM-dd}");
                }

                if (config.MaxDate.HasValue && dateValue > config.MaxDate.Value)
                {
                    result.AddError(location, $"Date {dateValue:yyyy-MM-dd} is after maximum allowed date of {config.MaxDate.Value:yyyy-MM-dd}");
                }
            }

            private void ValidateBoolean(string value, string location, ValidationResult result)
            {
                var validValues = new[] { "true", "false", "yes", "no", "1", "0" };
                if (!validValues.Contains(value.ToLowerInvariant()))
                {
                    result.AddError(location, $"Value '{value}' is not a valid boolean (accepted: true/false, yes/no, 1/0)");
                }
            }

            // FORMAT VALIDATIONS - Email
            //private void ValidateEmail(string value, ColumnConfig config, string location, ValidationResult result)
            //{
            //    var emailRegex = @"^[^@\s]+@[^@\s]+\.[^@\s]+$";
            //    if (!Regex.IsMatch(value, emailRegex))
            //    {
            //        result.AddError(location, $"Value '{value}' is not a valid email address");
            //    }

            //    if (config.MaxLength.HasValue && value.Length > config.MaxLength.Value)
            //    {
            //        result.AddError(location, $"Email length exceeds maximum allowed length of {config.MaxLength.Value}");
            //    }
            //}

            // FORMAT VALIDATIONS - Phone numbers
            //private void ValidatePhone(string value, string location, ValidationResult result)
            //{
            //    var digitsOnly = Regex.Replace(value, @"[\s\-\(\)\+]", "");

            //    if (!Regex.IsMatch(digitsOnly, @"^\d{7,15}$"))
            //    {
            //        result.AddError(location, $"Value '{value}' is not a valid phone number");
            //    }
            //}

            #endregion

            #region ROW-LEVEL VALIDATIONS

            // At least minimum required rows with data
            private void ValidateRowCount(IXLWorksheet worksheet, ExcelValidationConfig config, ValidationResult result)
            {
                var dataRowCount = worksheet.RowsUsed()
                    .Skip(config.DataStartRow - 1)
                    .Count(row => !IsRowEmpty(row));

                if (dataRowCount < config.MinimumDataRows)
                {
                    result.AddError("Data", $"File contains only {dataRowCount} data row(s), but minimum required is {config.MinimumDataRows}");
                }
            }

            private bool IsRowEmpty(IXLRow row)
            {
                return row.CellsUsed().All(c => string.IsNullOrWhiteSpace(c.GetString()));
            }

            #endregion
        }
        [HttpPost]
        public ActionResult ImportOldBoxesArchive(IFormFile excelFile)
        {
            string correlationId = Guid.NewGuid().ToString();

            try
            {
                //if (!ExcelValidationService(excelFile, out string message))
                //{
                //    return Json(new { isSuccess = false, message = message, correlationId = correlationId });
                //}
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
                        MaxLength = 11
                    },
                    new ColumnConfig
                    {
                        Name = "CompanyName",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        MaxLength = 22
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
                        //MustBeUnique = false,
                        MaxLength = 50
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
                        MaxLength = 1000
                    },
                    new ColumnConfig
                    {
                        Name = "BoxStatus",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        MaxLength = 10
                    },
                    new ColumnConfig
                    {
                        Name = "ArchivingPeriod",
                        DataType = CellDataType.Number,
                        IsRequired = true,
                        MinValue = -1,
                        MaxValue = 99,
                        MaxDecimalPlaces = 0
                    },
                    new ColumnConfig
                    {
                        Name = "BoxSentBy",
                        DataType = CellDataType.Text,
                        IsRequired = false,
                        MaxLength = 250
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

                // Use the injected service -- Validate the file
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

                // IMPORTANT: Copy file to memory stream to allow re-reading
                byte[] fileBytes;
                using (var memoryStream = new MemoryStream())
                {
                    excelFile.CopyTo(memoryStream);
                    fileBytes = memoryStream.ToArray();
                }

                // Create new IFormFile from bytes for GetOldBoxes
                var reReadableFile = new FormFile(
                    new MemoryStream(fileBytes),
                    0,
                    fileBytes.Length,
                    excelFile.Name,
                    excelFile.FileName)
                {
                    Headers = excelFile.Headers,
                    ContentType = excelFile.ContentType
                };

                // Now read the data
                ImportOldBoxesReq req = new ImportOldBoxesReq()
                {
                    BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                    OldBoxesList = GetOldBoxes(reReadableFile)
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
                    message = ex.Message, // Changed from ex.ToString() for cleaner message
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

                    string[] requiredColumns = new[]
                    {
                    "Code",
                    "CompanyName",
                    "IsActive",
                    "BoxRef",
                    "FileName",
                    "FileAdditionalInfo",
                    "BoxStatus",
                    "ArchivingPeriod",
                    "BoxSentBy",
                    "BoxSentDate",
                    "LastIndex"
                };

                    foreach (string col in requiredColumns)
                    {
                        if (!headers.ContainsKey(col))
                        {
                            throw new Exception($"Missing required column: [{col}]");
                        }
                    }

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
    }

    ===Archiving Controller Back End
            #region ExportWarehouseContainers
        [HttpPost]
        [Route("ExportWarehouseContainers")]
        public ExportWarehouseContainersRes ExportWarehouseContainers(ExportWarehouseContainersReq req)
        {
            ExportWarehouseContainersRes response = new ExportWarehouseContainersRes()
            {
                Req = req,
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = req.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ExportWarehouseContainers",
                UserName = req.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(req.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : req.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(req.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : req.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(req.BaseReq.CurrentEntity) && String.IsNullOrEmpty(req.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(ExportWarehouseContainersReq.BaseReq.CurrentEntity)} and {nameof(ExportWarehouseContainersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(req.BaseReq.CurrentEntity) ? String.Empty : req.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(req.BaseReq.CurrentBranch) ? String.Empty : req.BaseReq.CurrentBranch;

                LogInfo("ExportWarehouseContainers Has been called with the following Request", correlationInfo);
                LogInfoJson(req, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(req) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of ExportWarehouseContainers call", correlationInfo);

                    response.Resp = oBLL.ExportWarehouseContainers(req);

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("ExportWarehouseContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the ExportWarehouseContainers is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : req.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : req.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : req.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : req.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;


                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo("ExportWarehouseContainers Has Replied with the Following response", correlationInfo);
                LogInfoJson(response, correlationInfo);
                LogInfo("Calling the ExportWarehouseContainers is completed", correlationInfo);


                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = req.BaseReq.CorrelationId!;
                response.WebResp.User = req.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.Message;

                //this was added in case correlation Id was invalid(null or Empty)
                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                //this was added in case Username was invalid(null or Empty)
                correlationInfo.UserName = response.WebResp.User;

                //don't forget to change status code in case of exception
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = req.BaseReq.CorrelationId!;
                response.WebResp.User = req.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.Message;

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region ImportOldBoxes
        [HttpPost]
        [Route("ImportOldBoxes")]
        public ImportOldBoxesRes ImportOldBoxes(ImportOldBoxesReq req)
        {
            string correlationId = Guid.NewGuid().ToString();

            ImportOldBoxesRes response = new ImportOldBoxesRes()
            {
                Req = req,
                WebResp = new BaseResponse()
                {
                    CorrelationId = req.BaseReq.CorrelationId ?? throw new ArgumentNullException(nameof(req.BaseReq.CorrelationId)),
                    HttpResponseCode = HttpStatusCode.OK,
                    ResponseMessage = string.Empty,
                    User = req.BaseReq.CurrentUser ?? throw new ArgumentNullException(nameof(req.BaseReq.CurrentUser)),
                    Branch = req.BaseReq.CurrentBranch ?? throw new ArgumentNullException(nameof(req.BaseReq.CurrentBranch)),
                    Entity = req.BaseReq.CurrentEntity ?? throw new ArgumentNullException(nameof(req.BaseReq.CurrentEntity))
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = req.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ImportOldBoxes",
                UserName = req.BaseReq.CurrentUser
            };

            try
            {
                correlationInfo.Reserved = "ImportOldBoxes has been called with the following Request";
                LogInfoJson(req, correlationInfo);
             
                using (BLL.BLL oBLL = new(req.BaseReq.CurrentUser))
                {
                    oBLL.ImportOldBoxes(req);

                    response.WebResp.CorrelationId = req.BaseReq.CorrelationId;
                    response.WebResp.User = req.BaseReq.CurrentUser;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;
                    correlationInfo.Reserved = "ImportOldBoxes Has Replied with the Following response";
                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfoJson(response, correlationInfo);

                    return response;
                }
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = string.IsNullOrWhiteSpace(req.BaseReq.CorrelationId) ? correlationId : req.BaseReq.CorrelationId;
                response.WebResp.User = !string.IsNullOrWhiteSpace(req.BaseReq.CurrentUser) ? req.BaseReq.CurrentUser : "BadUser";
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;

                correlationInfo.Reserved = ex.Message;
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;
                correlationInfo.RDirection = RequestDirection.Response;

                LogErrorJson(ex, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = string.IsNullOrWhiteSpace(req.BaseReq.CorrelationId) ? correlationId : req.BaseReq.CorrelationId;
                response.WebResp.User = !string.IsNullOrWhiteSpace(req.BaseReq.CurrentUser) ? req.BaseReq.CurrentUser : "BadUser";
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;

                correlationInfo.Reserved = ex.Message;
                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogErrorJson(ex, correlationInfo);

                return response;
            }
        }
        #endregion

        Event
        ================
                private void BLL_OnPreEventEditContainerStatus(ref EditContainerStatusReq editContainerStatusReq)
        {
            if (editContainerStatusReq.StatusCode.Equals(ContainerStatusCode.SENT.ToString()) && !String.IsNullOrEmpty(editContainerStatusReq.Code))
            {
                editContainerStatusReq.PDF = String.Empty;
                String Entity = GetActiveEntity(editContainerStatusReq.HoldingEntityCode);

                GetContainerFilesReq getContainerFilesReq = new()
                {
                    BaseReq = editContainerStatusReq.BaseReq,
                    ContainerId = editContainerStatusReq.ContainerId
                };

                List<ArchivedFile> files = [];
                files = GetContainerFiles(getContainerFilesReq).Files;
                if (files.Count > 0 || editContainerStatusReq.Code is not null)
                {
                    Boolean Unlimited = false;
                    DateTime ArchivePeriod = DateTime.Now;
                    ArchivePeriod = ArchivePeriod.AddYears(files[0].ArchivingPeriod);

                    if (files[0].ArchivingPeriod == -1)
                    {
                        Unlimited = true;
                    }

                    if (files[0].CustomerId != null)
                    {
                        CustomerDocRequest customerDocRequest = new()
                        {
                            DestructionDate = Unlimited? "Unlimited":$"{ArchivePeriod:dd/MM/yyyy}",
                            ContainerID = editContainerStatusReq.Code!,
                            Entity = Entity,
                            User = editContainerStatusReq.BaseReq.CurrentUser!,
                            CustomerFiles = [],
                            CreationDate = $"{DateTime.Now:dd/MM/yyyy}"
                        };

                        Dictionary<String,List<String?>> fileDict=[];
                        foreach (ArchivedFile item in files)
                        {
                            if (!fileDict.ContainsKey(item.Name))
                            {
                                fileDict.Add(item.Name, [item.CustomerId!.ToString()]);
                            }
                            else
                            {
                                fileDict[item.Name].Add(item.CustomerId!.ToString());
                            }
                        }
                        foreach (KeyValuePair<String, List<String?>> DictEntry in fileDict)
                        {
                            customerDocRequest.CustomerFiles.Add(new() { DocumentType = DictEntry.Key, Id = DictEntry.Value! });
                        }
                        try
                        {
                            String data=JsonConvert.SerializeObject(customerDocRequest);
                            HttpContent content = new StringContent(data,Encoding.UTF8,"application/json");
                            HttpClient client = new();
                            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"]??throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                            Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}GenerateCustomerDocPDFForArchive", content);

                            Request.Wait();
                            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                            responseString.Wait();
                            editContainerStatusReq.PDF = responseString.Result;
                        }
                        catch (Exception ex)
                        {
                            throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
                        }

                    }
                    else if (files[0].CompanyCode.StartsWith("LB"))
                    {
                        BranchDocRequest branchDocRequest=new()
                        {
                            DestructionDate= Unlimited? "Unlimited":$"{ArchivePeriod:dd/MM/yyyy}",
                            ContainerID=editContainerStatusReq.Code!,
                            Entity=Entity,
                            User =editContainerStatusReq.BaseReq.CurrentUser!,
                            BranchFiles=[],
                            CreationDate = $"{DateTime.Now:dd/MM/yyyy}"
                        };
                        foreach (ArchivedFile item in files)
                        {
                            branchDocRequest.BranchFiles.Add(new()
                            {
                                DocumentType = item.Name,
                                FromDate = $"{item.FromDate:dd-MM-yyyy}",
                                ToDate = $"{item.ToDate:dd-MM-yyyy}"
                            });
                        }
                        try
                        {
                            String data=JsonConvert.SerializeObject(branchDocRequest);
                            HttpContent content = new StringContent(data,Encoding.UTF8,"application/json");
                            HttpClient client = new();
                            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"]??throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                            Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}GenerateBranchDocPDFForArchive", content);

                            Request.Wait();
                            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                            responseString.Wait();
                            editContainerStatusReq.PDF = responseString.Result;
                        }
                        catch (Exception ex)
                        {
                            throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
                        }

                    }
                    else if (files[0].CompanyCode.StartsWith("ET"))
                    {
                        EntityDocRequest entityDocRequest = new()
                        {
                            DestructionDate = Unlimited? "Unlimited":$"{ArchivePeriod:dd/MM/yyyy}",
                            ContainerID = editContainerStatusReq.Code!,
                            Entity = files[0].CompanyCode,
                            User = editContainerStatusReq.BaseReq.CurrentUser!,
                            EntityFiles = [],
                            CreationDate = $"{DateTime.Now:dd/MM/yyyy}"
                        };
                        foreach (ArchivedFile item in files)
                        {
                            entityDocRequest.EntityFiles.Add(new()
                            {
                                DocumentType = item.Name,
                                DocumentDescription = item.AdditionalInfo ?? String.Empty
                            });
                        }
                        try
                        {
                            String data = JsonConvert.SerializeObject(entityDocRequest);
                            HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
                            HttpClient client = new();
                            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ?? throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                            Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}GenerateEntityDocPDFForArchive", content);

                            Request.Wait();
                            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                            responseString.Wait();
                            editContainerStatusReq.PDF = responseString.Result;
                        }
                        catch (Exception ex)
                        {
                            throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
                        }
                    }
                }
                else
                {
                    throw new SGBLBadRequestException($"The Container With Sequence {editContainerStatusReq.Code} Dose Not Contain Any Files\n\rPlease Contact Support");
                }
                if (String.IsNullOrEmpty(editContainerStatusReq.PDF))
                {
                    throw new SGBLInternalServerException("PDF Service Malfunction");
                }
            }
        }

        private void BLL_OnPostEventEditContainerStatus(ref Container Container, ref EditContainerStatusReq editContainerStatusReq)
        {
            if (!String.IsNullOrEmpty(editContainerStatusReq.PDF))
            {
                Container.PDF = editContainerStatusReq.PDF;
            }
        }

        ====SQL sp
        USE [Alterna.Archive]
GO
/****** Object:  UserDefinedTableType [dbo].[TVP_Old_Boxes]    Script Date: 07/11/2025 11:23:22 AM ******/
CREATE TYPE [dbo].[TVP_Old_Boxes] AS TABLE(
	[RowId] [int] NOT NULL,
	[Code] [nvarchar](11) NULL,
	[CompanyName] [nvarchar](22) NULL,
	[Mnemonic] [nvarchar](50) NULL,
	[IsActive] [bit] NULL,
	[BoxRef] [nvarchar](50) NULL,
	[FileName] [nvarchar](250) NULL,
	[AdditionalInfo] [nvarchar](1000) NULL,
	[StatusCode] [nvarchar](10) NULL,
	[ArchivingPeriod] [int] NULL,
	[BoxSentBy] [nvarchar](250) NULL,
	[BoxSentDate] [datetime] NULL,
	[LastIndex] [bigint] NULL
)
GO
/****** Object:  Table [dbo].[Configuration]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[Configuration](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[SettingName] [nvarchar](50) NOT NULL,
	[SettingValue] [nvarchar](50) NOT NULL,
	[IsActive] [bit] NOT NULL,
	[SettingDescription] [nvarchar](1000) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_Configuration] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[lkp_FileType]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[lkp_FileType](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](10) NOT NULL,
	[Entity] [nvarchar](10) NOT NULL,
	[Category] [nvarchar](10) NOT NULL,
	[Description] [nvarchar](250) NOT NULL,
	[HasDate] [bit] NOT NULL,
	[IsCustomer] [bit] NOT NULL,
	[ArchivingPeriod] [int] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_lkp_FileType] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[lkp_Status]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[lkp_Status](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](10) NOT NULL,
	[Description] [nvarchar](100) NULL,
	[Category] [nvarchar](10) NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_lkp_Status] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Container]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Container](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](50) NOT NULL,
	[CompanyCode] [nvarchar](9) NOT NULL,
	[Entity] [nvarchar](10) NOT NULL,
	[CurrentLocation] [nvarchar](50) NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[ArchivingDate] [datetime] NULL,
	[isDeleted] [bit] NOT NULL,
	[isNotified] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Container] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_ContainerStatus]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_ContainerStatus](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[ContainerId] [int] NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[HoldingEntityCode] [nvarchar](11) NOT NULL,
	[isCurrentStatus] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_ContainerStatus] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_CurrentContainerFileRelationship]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_CurrentContainerFileRelationship](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[FileId] [int] NOT NULL,
	[ContainerId] [int] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_CurrentContainerFileRelationship] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Customer]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Customer](
	[Id] [int] NOT NULL,
	[CompanyBook] [nvarchar](11) NULL,
	[ShortName] [nvarchar](max) NULL,
	[GivenNames] [nvarchar](max) NULL,
	[FamilyName] [nvarchar](max) NULL,
	[FatherName] [nvarchar](max) NULL,
	[MoFirstName] [nvarchar](max) NULL,
	[MoLastName] [nvarchar](max) NULL,
	[LegalId] [nvarchar](max) NULL,
	[PCntryCode] [nvarchar](max) NULL,
	[PhoneAreaCode] [nvarchar](max) NULL,
	[PhoneNo] [nvarchar](max) NULL,
	[MCntryCode] [nvarchar](max) NULL,
	[LbmbAreaCode] [nvarchar](max) NULL,
	[LbmbMobilenb] [nvarchar](max) NULL,
	[AddCustType] [nvarchar](max) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Client] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_File]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_File](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[CustomerId] [int] NULL,
	[Name] [nvarchar](250) NOT NULL,
	[FileTypeCode] [nvarchar](10) NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[CompanyCode] [nvarchar](9) NOT NULL,
	[FromDate] [date] NULL,
	[ToDate] [date] NULL,
	[AdditionalInfo] [nvarchar](1000) NULL,
	[isDeleted] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_File] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_FileStatus]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_FileStatus](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[FileId] [int] NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[HoldingEntityCode] [nvarchar](11) NOT NULL,
	[isCurrentStatus] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_FileStatus] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_PDF]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_PDF](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[PDF] [varbinary](max) NOT NULL,
	[Request] [nvarchar](max) NOT NULL,
	[ApiMethod] [nvarchar](500) NOT NULL,
	[BranchList] [nvarchar](max) NULL,
	[Entity] [nvarchar](10) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_PDF] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Sequence]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Sequence](
	[SequenceId] [int] IDENTITY(1,1) NOT NULL,
	[Owner] [nvarchar](50) NOT NULL,
	[Prefix] [nvarchar](50) NULL,
	[LastIndex] [bigint] NOT NULL,
	[Suffix] [nvarchar](50) NULL,
	[IsActive] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Sequence] PRIMARY KEY CLUSTERED 
(
	[SequenceId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET IDENTITY_INSERT [dbo].[Configuration] ON 

INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (1, N'BranchBoxMin', N'1', 0, N'Minimum number of branch files per box', N'ArchivingInit', CAST(N'2024-06-10T10:55:52.973' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:52.973' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (2, N'BranchBoxMax', N'4', 1, N'Maximum number of branch files per box', N'ArchivingInit', CAST(N'2024-06-10T10:55:52.980' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:52.980' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (7, N'EntityBoxMin', N'1', 1, N'Minimum number of entity files per box', N'ArchivingInit', CAST(N'2024-06-10T10:55:52.987' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:52.987' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (8, N'EntityBoxMax', N'5', 1, N'Maximum number of entity files per box', N'ArchivingInit', CAST(N'2024-06-10T10:55:52.990' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:52.990' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (9, N'FileInfoFreeTextMaxLength', N'200', 1, N'Maximum number of characters for "file info" per file', N'ArchivingInit', CAST(N'2024-06-10T10:55:53.003' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:53.003' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (10, N'CustomerBoxMin', N'5', 0, N'Minimum number of customer files per box', N'ArchivingInit', CAST(N'2024-06-10T10:55:53.010' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:53.010' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (11, N'CustomerBoxMax', N'7', 1, N'Maximum number of customer files per box', N'ArchivingInit', CAST(N'2024-06-10T10:55:53.017' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:53.017' AS DateTime))
INSERT [dbo].[Configuration] ([Id], [SettingName], [SettingValue], [IsActive], [SettingDescription], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (12, N'UniqueCustomerID', N'0', 1, N'Can The Box Have Duplicate Customer Id', N'ArchivingInit', CAST(N'2024-06-10T10:55:53.020' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:53.020' AS DateTime))
SET IDENTITY_INSERT [dbo].[Configuration] OFF
SET IDENTITY_INSERT [dbo].[lkp_FileType] ON 

INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (1, N'1', N'', N'Branch', N'Client Folder', 0, 1, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.657' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.657' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (2, N'2', N'', N'Branch', N'Chèques retournés impayés', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.663' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.663' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (3, N'3', N'', N'Branch', N'Régularisation chèques', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.667' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.667' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (4, N'4', N'', N'Branch', N'Relevé de compte courant / Emission d’un relevé de compte epargne ou octroi d’une copie conforme d’un livret d’epargne ', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.670' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.670' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (5, N'5', N'', N'Branch', N'Confirmation de solde', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.673' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.673' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (6, N'6', N'', N'Branch', N'Attestation', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.677' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.677' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (7, N'7', N'', N'Branch', N'Avis de recherche', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.680' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.680' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (8, N'8', N'', N'Branch', N'Demande carte', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.687' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.687' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (9, N'9', N'', N'Branch', N'Réception carte et PIN', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.690' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.690' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (10, N'10', N'', N'Branch', N'Activation carte', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.693' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.693' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (11, N'11', N'', N'Branch', N'PIN reorder', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.700' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.700' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (12, N'12', N'', N'Branch', N'Remplacement carte', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.703' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.703' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (13, N'13', N'', N'Branch', N'Annulation carte', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.707' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.707' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (14, N'14', N'', N'Branch', N'Instructions générales', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.713' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.713' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (15, N'15', N'', N'Branch', N'Réclamation carte', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.717' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.717' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (16, N'16', N'', N'Branch', N'Carte retenue ATM', 1, 0, 4, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.720' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.720' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (17, N'17', N'', N'Branch', N'Livret égaré', 1, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:54:26.723' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.723' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (18, N'18', N'ET9900067', N'Not Branch', N'Credit files', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.083' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.083' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (19, N'19', N'ET9900127', N'Not Branch', N'Correspondent Contract & Doc', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.090' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.090' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (20, N'20', N'ET9900127', N'Not Branch', N'Counterparties Limits', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.097' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.097' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (21, N'21', N'ET9900127', N'Not Branch', N'KYC forms', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.103' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.103' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (22, N'22', N'ET9900049', N'Not Branch', N'Position de change journalière', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.117' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.117' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (23, N'23', N'ET9900106', N'Not Branch', N'Dossier commercial non utilisé', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.123' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.123' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (24, N'24', N'ET9900106', N'Not Branch', N'Dossier clôturé auprès de REAL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.127' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.127' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (25, N'25', N'ET9900106', N'Not Branch', N'Doss.clôturé action judiciaire', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.133' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.133' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (26, N'26', N'ET9900106', N'Not Branch', N'Documents divers', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.137' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.137' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (27, N'27', N'ET9900111', N'Not Branch', N'Contrats baux originaux', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.140' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.140' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (28, N'28', N'ET9900111', N'Not Branch', N'Reçus des taxes locatives', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.143' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.143' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (29, N'29', N'ET9900111', N'Not Branch', N'Reçus des taxes financières', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.147' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.147' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (30, N'30', N'ET9900111', N'Not Branch', N'Frais communs - loyers', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.150' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.150' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (31, N'31', N'ET9900111', N'Not Branch', N'Dossiers BFs en dations vendus', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.153' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.153' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (32, N'32', N'ET9900111', N'Not Branch', N'Comités de vente', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.160' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.160' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (33, N'33', N'ET9900111', N'Not Branch', N'Demandes de paiement', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.163' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.163' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (34, N'34', N'ET9900111', N'Not Branch', N'Taxes et impôts', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.170' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.170' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (35, N'35', N'ET9900111', N'Not Branch', N'Attest. Municipale-financière', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.173' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.173' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (36, N'36', N'ET9900038', N'Not Branch', N'Liste chèques retournés impayé', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.180' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.180' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (37, N'37', N'ET9900038', N'Not Branch', N'Chèques à régulariser', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.183' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.183' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (38, N'38', N'ET9900128', N'Not Branch', N'LG cancelled', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.187' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.187' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (39, N'39', N'ET9900128', N'Not Branch', N'LC import & export  cancelled', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.190' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.190' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (40, N'40', N'ET9900128', N'Not Branch', N'RD import & export cancelled', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.197' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.197' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (41, N'41', N'ET9900128', N'Not Branch', N'SP & NARVAL- statistics', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.200' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.200' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (42, N'42', N'ET9900128', N'Not Branch', N'CD  Cancelled', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.203' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.203' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (43, N'43', N'ET9900066', N'Not Branch', N'Prêts/emprunts interbancaires', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.210' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.210' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (44, N'44', N'ET9900066', N'Not Branch', N'Liste prêt/emprunt en cours', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.213' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.213' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (45, N'45', N'ET9900066', N'Not Branch', N'Virements cpte NOSTRO/VOSTRO', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.220' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.220' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (46, N'46', N'ET9900066', N'Not Branch', N'Opération change', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.223' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.223' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (47, N'47', N'ET9900066', N'Not Branch', N'Position cptes Clients spec.', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.227' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.227' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (48, N'48', N'ET9900066', N'Not Branch', N'Transfert FIDUS faveur clients', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.230' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.230' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (49, N'49', N'ET9900066', N'Not Branch', N'Opérations Achat/vente titres', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.237' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.237' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (50, N'50', N'ET9900066', N'Not Branch', N'Paiement coupon et dividendes', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.240' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.240' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (51, N'51', N'ET9900066', N'Not Branch', N'Opérations des droits de garde', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.243' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.243' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (52, N'52', N'ET9900066', N'Not Branch', N'Réconciliation dépositaire', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.250' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.250' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (53, N'53', N'ET9900066', N'Not Branch', N'Taxes + intérêts interbancaire', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.253' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.253' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (54, N'54', N'ET9900066', N'Not Branch', N'BT,Eurobonds...Faveur clients', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.257' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.257' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (55, N'55', N'ET9900066', N'Not Branch', N'BT,Eurobonds...Faveur SGBL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.263' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.263' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (56, N'56', N'ET9900066', N'Not Branch', N'Modifications Taux', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.267' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.267' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (57, N'57', N'ET9900066', N'Not Branch', N'Liste des Transactions FIDUS', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.270' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.270' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (58, N'58', N'ET9900066', N'Not Branch', N'DDG payés faveur dépositaires', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.277' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.277' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (59, N'59', N'ET9900066', N'Not Branch', N'Remat actions SOLIDERE', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.280' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.280' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (60, N'60', N'ET9900066', N'Not Branch', N'Retrait + Transfert Titres', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.283' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.283' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (61, N'61', N'ET9900066', N'Not Branch', N'Circulaire BDL 91', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.290' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.290' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (62, N'62', N'ET9900066', N'Not Branch', N'Virement XAU/XAG faveur Client', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.293' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.293' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (63, N'63', N'ET9900066', N'Not Branch', N'Ouv. /ferm. cpte NOSTRO/VOSTRO', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.300' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.300' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (64, N'64', N'ET9900065', N'Not Branch', N'Bordereaux prime hospit.', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.303' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.303' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (65, N'65', N'ET9900065', N'Not Branch', N'Bordereaux prime vie ', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.310' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.310' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (66, N'66', N'ET9900065', N'Not Branch', N'Recup. Fact. Ass. Hospit.', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.313' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.313' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (67, N'67', N'ET9900065', N'Not Branch', N'Avenant assur. Hospitalisation', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.320' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.320' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (68, N'68', N'ET9900065', N'Not Branch', N'Avenant assurance vie', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.323' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.323' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (69, N'69', N'ET9900065', N'Not Branch', N'Instructions terminees', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.330' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.330' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (70, N'70', N'ET9900065', N'Not Branch', N'Contrat hospitalisation expire', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.333' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.333' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (71, N'71', N'ET9900065', N'Not Branch', N'Contrats vie expires', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.337' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.337' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (72, N'72', N'ET9900065', N'Not Branch', N'Assurance Expatries', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.343' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.343' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (73, N'73', N'ET9900065', N'Not Branch', N'Accident de travail', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.347' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.347' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (74, N'74', N'ET9900065', N'Not Branch', N'Sinistre vie', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.350' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.350' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (75, N'85', N'ET9900065', N'Not Branch', N'Divers CNSS & Assurance', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.357' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.357' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (76, N'86', N'ET9900065', N'Not Branch', N'Recuperation  Cheque CNSS', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.360' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.360' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (77, N'87', N'ET9900083', N'Not Branch', N'Déclaration BCL Circular 158', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.363' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.363' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (78, N'88', N'ET9900093', N'Not Branch', N'Procès défendeurs/offenseurs', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.370' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.370' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (79, N'89', N'ET9900093', N'Not Branch', N'Dévolutions successorales', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.370' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.370' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (80, N'90', N'ET9900093', N'Not Branch', N'Surveillance permanente', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.377' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.377' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (81, N'91', N'ET9900027', N'Not Branch', N'OP RECU $', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.380' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.380' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (82, N'92', N'ET9900027', N'Not Branch', N'OP RECU LBP', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.387' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.387' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (83, N'93', N'ET9900027', N'Not Branch', N'OP RECU DEVISES', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.390' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.390' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (84, N'94', N'ET9900027', N'Not Branch', N'OP ENV $ ETRANGER', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.393' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.393' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (85, N'95', N'ET9900027', N'Not Branch', N'OP ENV EUR ETRANGER', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.400' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.400' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (86, N'96', N'ET9900027', N'Not Branch', N'OP ENV DEVISES ETRANGER', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.403' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.403' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (87, N'97', N'ET9900027', N'Not Branch', N'OP ENV $ LOCAL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.410' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.410' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (88, N'98', N'ET9900027', N'Not Branch', N'OP ENV LBP', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.413' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.413' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (89, N'99', N'ET9900027', N'Not Branch', N'OP ENV DEVISES LOCAL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.420' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.420' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (90, N'100', N'ET9900027', N'Not Branch', N'DIVERS(CAC, RETOUR...)', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.423' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.423' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (91, N'101', N'ET9900013', N'Not Branch', N'CHEQUES A L''ETRANGER - PAYES', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.430' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.430' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (92, N'102', N'ET9900013', N'Not Branch', N'CHEQUES A L''ETRANGER -  RECUS', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.433' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.433' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (93, N'103', N'ET9900013', N'Not Branch', N'CHEQUES A L''ETRANGER - EN OPP', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.440' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.440' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (94, N'104', N'ET9900013', N'Not Branch', N'SALAIRE SEC PRIVE', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.443' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.443' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (95, N'105', N'ET9900013', N'Not Branch', N'SALAIRE SEC PUB', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.447' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.447' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (96, N'106', N'ET9900013', N'Not Branch', N'FACTURES', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.450' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.450' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (97, N'107', N'ET9900013', N'Not Branch', N'MEC,TAX & TVA', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.457' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.457' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (98, N'108', N'ET9900013', N'Not Branch', N'EFFETS EPH', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.460' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.460' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (99, N'109', N'ET9900013', N'Not Branch', N'CP 158', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.463' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.463' AS DateTime))
GO
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (100, N'110', N'ET9900086', N'Not Branch', N'Fidus FX Position', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.467' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.467' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (101, N'111', N'ET9900047', N'Not Branch', N'Start of day list', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.470' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.470' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (102, N'112', N'ET9900047', N'Not Branch', N'Debit/Credit Operations', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.477' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.477' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (103, N'113', N'ET9900047', N'Not Branch', N'List of daily transactions', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.480' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.480' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (104, N'114', N'ET9900047', N'Not Branch', N'End of day List', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.487' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.487' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (105, N'115', N'ET9900047', N'Not Branch', N'Outward transfers', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.490' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.490' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (106, N'116', N'ET9900047', N'Not Branch', N'Clients / Divers', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.493' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.493' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (107, N'117', N'ET9900010', N'Not Branch', N'Audit reports', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.500' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.500' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (108, N'118', N'ET9900099', N'Not Branch', N'Contrats', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.503' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.503' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (109, N'119', N'ET9900099', N'Not Branch', N'Demande d''engagement', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.507' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.507' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (110, N'120', N'ET9900004', N'Not Branch', N'Audit financial statements', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.513' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.513' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (111, N'121', N'ET9900004', N'Not Branch', N'Signed BCC-BDL report-ehibit', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.517' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.517' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (112, N'124', N'ET9900000', N'Not Branch', N'Dossier de Prêt', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.520' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.520' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (113, N'125', N'', N'Not Branch', N'Annulation Garanties', 1, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.523' AS DateTime), N'clababidi', CAST(N'2025-09-25T09:27:51.030' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (114, N'126', N'ET9900000', N'Not Branch', N'Ouverture de compte', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.530' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.530' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (115, N'127', N'ET9900000', N'Not Branch', N'Cartes bancaires (dem/recep.)', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.533' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.533' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (116, N'128', N'ET9900000', N'Not Branch', N'Chéquiers (demande/recepissé)', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.537' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.537' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (117, N'129', N'ET9900000', N'Not Branch', N'Domiciliations Factures Div.', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.543' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.543' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (118, N'130', N'ET9900000', N'Not Branch', N'KYC', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.547' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.547' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (119, N'131', N'ET9900000', N'Not Branch', N'Financial Market', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.553' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.553' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (120, N'132', N'ET9900000', N'Not Branch', N'Circulaire 158', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.557' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.557' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (121, N'133', N'ET9900054', N'Not Branch', N'Demande d''engagement', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.560' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.560' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (122, N'134', N'ET9900054', N'Not Branch', N'Attestations diverses', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.567' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.567' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (123, N'135', N'ET9900054', N'Not Branch', N'Dossiers Subventions Scolaires', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.570' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.570' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (124, N'136', N'ET9900054', N'Not Branch', N'Mission à l''Etranger', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.577' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.577' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (125, N'137', N'ET9900054', N'Not Branch', N'Ordre de Mission', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.580' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.580' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (126, N'138', N'ET9900054', N'Not Branch', N'Congé sans solde', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.583' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.583' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (127, N'139', N'ET9900054', N'Not Branch', N'Solde Négatif', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.590' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.590' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (128, N'140', N'ET9900054', N'Not Branch', N'Déduction Congé Maladie', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.593' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.593' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (129, N'141', N'ET9900054', N'Not Branch', N'Rapports Médicaux', 0, 0, 2, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.597' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.597' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (130, N'142', N'ET9900054', N'Not Branch', N'Heures Supplémentaires', 0, 0, 2, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.600' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.600' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (131, N'143', N'ET9900054', N'Not Branch', N'Fondations', 0, 0, 2, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.607' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.607' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (132, N'144', N'ET9900107', N'Not Branch', N'Etats CCB - BDL - Circulaire 7', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.610' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.610' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (133, N'145', N'ET9900107', N'Not Branch', N'Chrono', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.613' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.613' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (134, N'146', N'ET9900107', N'Not Branch', N'Filiales', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.620' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.620' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (135, N'147', N'ET9900107', N'Not Branch', N'Corresp. ABL-BDL-CCB-CMA-SIC', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.623' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.623' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (136, N'148', N'ET9900107', N'Not Branch', N'Contrats avocats', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.627' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.627' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (137, N'149', N'ET9900107', N'Not Branch', N'Frais généraux ', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.633' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.633' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (138, N'150', N'ET9900107', N'Not Branch', N'Narval - Audit Pro', 0, 0, 5, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.637' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.637' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (139, N'151', N'ET9900107', N'Not Branch', N'Projets divers (SAIFI 415...)-1', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.640' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.640' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (140, N'152', N'ET9900107', N'Not Branch', N'Contrats Prêts', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.643' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.643' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (141, N'153', N'ET9900107', N'Not Branch', N'Projets de développement', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.650' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.650' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (142, N'154', N'ET9900107', N'Not Branch', N'Circulaires BDL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.653' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.653' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (143, N'155', N'ET9900107', N'Not Branch', N'Midclear:registre-act.ordin.', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.660' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.660' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (144, N'156', N'ET9900107', N'Not Branch', N'Capital (augmentation-apports)', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.663' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.663' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (145, N'157', N'ET9900107', N'Not Branch', N'Statuts  ', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.667' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.667' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (146, N'158', N'ET9900107', N'Not Branch', N'Comités issus du Conseil', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.670' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.670' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (147, N'159', N'ET9900107', N'Not Branch', N'Comités internes spécialisés', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.687' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.687' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (148, N'160', N'ET9900107', N'Not Branch', N'Conseil Administrat.-Assemblée', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.690' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.690' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (149, N'161', N'ET9900107', N'Not Branch', N'Etats finan.-Rapports spéciaux', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.697' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.697' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (150, N'162', N'ET9900107', N'Not Branch', N'Commissaires Surveillance ', 1, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.700' AS DateTime), N'clababidi', CAST(N'2025-06-27T13:12:11.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (151, N'163', N'ET9900107', N'Not Branch', N'Missions CCB - CMA', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.707' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.707' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (152, N'164', N'ET9900107', N'Not Branch', N'Registre de Commerce', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.710' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.710' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (153, N'165', N'ET9900107', N'Not Branch', N'Signatures autorisées', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.713' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.713' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (154, N'166', N'ET9900107', N'Not Branch', N'Filiales - 2', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.717' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.717' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (155, N'167', N'ET9900107', N'Not Branch', N'Projets divers (SAIFI 415...)-2', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.720' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.720' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (156, N'168', N'ET9900016', N'Not Branch', N'Bon de transport - Mecattaf', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.727' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.727' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (157, N'169', N'ET9900016', N'Not Branch', N'Purchase order - Mecattaf', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.730' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.730' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (158, N'170', N'ET9900016', N'Not Branch', N'Accounting advices - Mecattaf', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.733' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.733' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (159, N'171', N'ET9900016', N'Not Branch', N'Deposit amount letters - BDL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.737' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.737' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (160, N'172', N'ET9900016', N'Not Branch', N'Copy cash withdrawal chqs-BDL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.743' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.743' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (161, N'173', N'ET9900016', N'Not Branch', N'Accounting entries advices-BDL', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.747' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.747' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (162, N'174', N'ET9900016', N'Not Branch', N'Transport vouchers - Prosec', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.750' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.750' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (163, N'175', N'ET9900016', N'Not Branch', N'ATM - Charging / Discharging', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.757' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.757' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (164, N'176', N'ET9900119', N'Not Branch', N'Mission Classique', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.760' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.760' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (165, N'177', N'ET9900119', N'Not Branch', N'Mission Investigation', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.767' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.767' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (166, N'178', N'ET9900119', N'Not Branch', N'Mission thématique', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.770' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.770' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (167, N'179', N'ET9900119', N'Not Branch', N'Mission Conseil', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.777' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.777' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (168, N'180', N'ET9900119', N'Not Branch', N'Documents travail- classique', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.780' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (169, N'181', N'ET9900119', N'Not Branch', N'Document travail-Investigation', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.783' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.783' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (170, N'182', N'ET9900119', N'Not Branch', N'Documents travail- thématique', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.790' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.790' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (171, N'183', N'ET9900119', N'Not Branch', N'Documents de travail- Conseil', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.793' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.793' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (172, N'184', N'ET9900119', N'Not Branch', N'Documents Direction', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.800' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.800' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (173, N'185', N'ET9900119', N'Not Branch', N'Avis de recherche', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.803' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.803' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (174, N'186', N'ET9900119', N'Not Branch', N'Comité d''Audit- SGBL', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.807' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.807' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (175, N'187', N'ET9900119', N'Not Branch', N'Comités BOARD', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.810' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.810' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (176, N'188', N'ET9900119', N'Not Branch', N'Comités de Direction', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.813' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.813' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (177, N'189', N'ET9900119', N'Not Branch', N'Comité d''Audit-Filiales', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:31.820' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:31.820' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (178, N'122', N'ET9900000', N'Not Branch', N'تصريح إسمي سنوي', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.233' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.233' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (179, N'123', N'ET9900000', N'Not Branch', N'جدول الأشتراكات', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.243' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.243' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (180, N'75', N'ET9900065', N'Not Branch', N'جداول الاستلام', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.260' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.260' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (181, N'76', N'ET9900065', N'Not Branch', N'تحاقيق اجتماعية', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.267' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.267' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (182, N'77', N'ET9900065', N'Not Branch', N'اعفاء عن التواقيع', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.270' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.270' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (183, N'78', N'ET9900065', N'Not Branch', N'تفويض', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.273' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.273' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (184, N'79', N'ET9900065', N'Not Branch', N'جداول الاشتراكات', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.280' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.280' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (185, N'80', N'ET9900065', N'Not Branch', N'ترك عمل', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.283' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.283' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (186, N'81', N'ET9900065', N'Not Branch', N'افادات جامعية', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.287' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.287' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (187, N'82', N'ET9900065', N'Not Branch', N'اعلام عن استخدام اجير', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.290' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.290' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (188, N'83', N'ET9900065', N'Not Branch', N'تصريح عن استخدام اجير', 0, 0, -1, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.293' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.293' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (189, N'84', N'ET9900065', N'Not Branch', N'نفقات دفن', 0, 0, 10, N'ArchivingInit', CAST(N'2024-06-10T10:55:37.297' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:55:37.297' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (191, N'190', N'ET9900130', N'Not Branch', N'Test1', 0, 0, 2, N'clababidi', CAST(N'2025-07-16T13:54:09.277' AS DateTime), N'clabbaidi', CAST(N'2025-07-16T13:54:09.277' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (192, N'191', N'ET9900131', N'Not Branch', N'File ACIN', 0, 0, 5, N'clababidi', CAST(N'2025-07-16T13:54:09.277' AS DateTime), N'clabbaidi', CAST(N'2025-07-16T13:54:09.277' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (319, N'192', N'ET9900111', N'Not Branch', N'Contrats baux originaux old boxes', 0, 0, -1, N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime), N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (320, N'193', N'ET9900111', N'Not Branch', N'File REES 1', 0, 0, -1, N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime), N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (321, N'194', N'ET9900111', N'Not Branch', N'File REES 3', 0, 0, -1, N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime), N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (322, N'195', N'ET9900132', N'Not Branch', N'File AFFA 1', 0, 0, 5, N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime), N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (323, N'196', N'ET9900132', N'Not Branch', N'File AFFA 2', 0, 0, 5, N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime), N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (324, N'197', N'ET9900132', N'Not Branch', N'File AFFA 3', 0, 0, 5, N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime), N'psammia', CAST(N'2025-09-30T10:34:42.417' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (325, N'198', N'ET9900099', N'Not Branch', N'File DEVE 1', 0, 0, 5, N'psammia', CAST(N'2025-10-24T11:42:24.887' AS DateTime), N'psammia', CAST(N'2025-10-24T11:42:24.887' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (326, N'199', N'ET9900099', N'Not Branch', N'File deve 2', 0, 0, 5, N'psammia', CAST(N'2025-10-24T11:42:24.887' AS DateTime), N'psammia', CAST(N'2025-10-24T11:42:24.887' AS DateTime))
GO
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (327, N'200', N'ET9900133', N'Not Branch', N'SAA File 1', 0, 0, 5, N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime), N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (328, N'201', N'ET9900133', N'Not Branch', N'SAA File 2', 0, 0, 5, N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime), N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (329, N'202', N'ET9900133', N'Not Branch', N'SAA File 3', 0, 0, 5, N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime), N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (330, N'203', N'ET9900133', N'Not Branch', N'SAA File 4', 0, 0, 5, N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime), N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (331, N'204', N'ET9900133', N'Not Branch', N'SAA File 5', 0, 0, 5, N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime), N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (332, N'205', N'ET9900133', N'Not Branch', N'SAA File 6', 0, 0, 5, N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime), N'clababidi', CAST(N'2025-10-28T12:09:11.523' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (448, N'206', N'ET9900004', N'Not Branch', N'File Regl 1', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:02:47.450' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:02:47.450' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (449, N'207', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2016', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (450, N'208', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2018', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (451, N'209', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2019', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (452, N'210', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2020', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (453, N'211', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2021 1-7', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (454, N'212', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2021 8-12', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (455, N'213', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2022', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (456, N'214', N'ET9900004', N'Not Branch', N'BCC ANNEXES 2024', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (457, N'215', N'ET9900004', N'Not Branch', N'BRGCC 2018-2019', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (458, N'216', N'ET9900004', N'Not Branch', N'BRGCC 2020', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (459, N'217', N'ET9900004', N'Not Branch', N'BRGCC 2021', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (460, N'218', N'ET9900004', N'Not Branch', N'BRGCC DIVERS DOC', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (461, N'219', N'ET9900004', N'Not Branch', N'CONFIRMATIONS BANKS 2014-2016', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (462, N'220', N'ET9900004', N'Not Branch', N'CONFIRMATIONS BANKS 2017-2022', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (463, N'221', N'ET9900004', N'Not Branch', N'DIVERS', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (464, N'222', N'ET9900004', N'Not Branch', N'LCB 2019 + DIVERS', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (465, N'223', N'ET9900004', N'Not Branch', N'LCB 3508-3510-3520', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (466, N'224', N'ET9900004', N'Not Branch', N'LCB DOC', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (467, N'225', N'ET9900004', N'Not Branch', N'LCB1', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (468, N'226', N'ET9900004', N'Not Branch', N'LCB2', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (469, N'227', N'ET9900004', N'Not Branch', N'LCB-IBH', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (470, N'228', N'ET9900004', N'Not Branch', N'LCB-SOLIDERE', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (471, N'229', N'ET9900004', N'Not Branch', N'SGLF 2002-2009 / COURTAGE 2003-2017', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (472, N'230', N'ET9900004', N'Not Branch', N'SGLF BOOKS', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (473, N'231', N'ET9900004', N'Not Branch', N'SGSI 2002-2010', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (474, N'232', N'ET9900004', N'Not Branch', N'SGSI 2008-2013', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (475, N'233', N'ET9900004', N'Not Branch', N'SGSI BOOKS', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (476, N'234', N'ET9900004', N'Not Branch', N'WP SUBSIDIARIES 2017-2018', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (477, N'235', N'ET9900004', N'Not Branch', N'WP SUBSIDIARIES 2018-2019-2020-2021', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (478, N'236', N'ET9900004', N'Not Branch', N'WP SUBSIDIARIES 2021-2022-2023-2024', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T10:49:20.780' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (479, N'237', N'ET9900004', N'Not Branch', N'File Regl 2', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-05T14:31:53.200' AS DateTime), N'AlternaSystem', CAST(N'2025-11-05T14:31:53.200' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (480, N'238', N'ET9900004', N'Not Branch', N'File Regl', 0, 0, 10, N'AlternaSystem', CAST(N'2025-11-06T14:29:22.190' AS DateTime), N'AlternaSystem', CAST(N'2025-11-06T14:29:22.190' AS DateTime))
INSERT [dbo].[lkp_FileType] ([Id], [Code], [Entity], [Category], [Description], [HasDate], [IsCustomer], [ArchivingPeriod], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (481, N'239', N'ET9900004', N'Not Branch', N'REGL 100', 0, 0, -1, N'AlternaSystem', CAST(N'2025-11-06T14:46:44.840' AS DateTime), N'AlternaSystem', CAST(N'2025-11-06T14:46:44.840' AS DateTime))
SET IDENTITY_INSERT [dbo].[lkp_FileType] OFF
SET IDENTITY_INSERT [dbo].[lkp_Status] ON 

INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (1, N'PENDING', N'Container created', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.733' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.733' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (2, N'SENTFORVAL', N'Container sent for validation by the  RCA', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.740' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.740' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (3, N'VALIDATED', N'Container status validate by the DA', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.740' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.740' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (4, N'SENT', N'Container sent to the warehouse', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.747' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.747' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (5, N'RECEIVED', N'Container received to the warehouse', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.750' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.750' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (6, N'TOBEDESTR', N'Container reaching the expiry date, waiting to be destroyed', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.753' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.753' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (7, N'DESTROYED', N'Container destroyed', N'CONTAINER', N'ArchivingInit', CAST(N'2024-06-10T10:54:26.757' AS DateTime), N'ArchivingInit', CAST(N'2024-06-10T10:54:26.757' AS DateTime))
INSERT [dbo].[lkp_Status] ([Id], [Code], [Description], [Category], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (8, N'NOTFOUND', N'Container old and not found', N'CONTAINER', N'ArchivingInit', CAST(N'2025-11-05T00:00:00.000' AS DateTime), N'ArchivingInit', CAST(N'2025-11-05T00:00:00.000' AS DateTime))
SET IDENTITY_INSERT [dbo].[lkp_Status] OFF
SET IDENTITY_INSERT [dbo].[t_Sequence] ON 

INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (1, N'LB0010007', N'HAMR.', 288, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.393' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.393' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (2, N'LB0010008', N'JBEI.', 51, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.393' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.393' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (3, N'LB0010010', N'TRIP.', 206, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.397' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.397' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (4, N'LB0010011', N'MAZR.', 223, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.397' AS DateTime), N'clababidi', CAST(N'2025-07-14T09:46:10.447' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (5, N'LB0010013', N'SASS.', 244, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.397' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.397' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (6, N'LB0010015', N'ZAHL.', 25, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.400' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.400' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (7, N'LB0010016', N'JOUN.', 121, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.400' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.400' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (8, N'LB0010017', N'CHTA.', 115, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.400' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.400' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (9, N'LB0010018', N'SAID.', 190, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.403' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.403' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (10, N'LB0010026', N'AMIO.', 247, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.403' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.403' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (11, N'LB0010063', N'ASAG.', 60, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.407' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.407' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (12, N'LB0010203', N'JDIB.', 272, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.407' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.407' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (13, N'LB0010200', N'SADK.', 466, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.407' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.407' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (14, N'LB0010206', N'TAKL.', 213, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.410' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.410' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (15, N'LB0010213', N'NAFA.', 186, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.410' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.410' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (16, N'LB0010214', N'SOMA.', 152, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.410' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.410' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (17, N'LB0010218', N'CHAR.', 119, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.413' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.413' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (18, N'LB0010236', N'MANS.', 163, NULL, 1, N'ArchivingInit', CAST(N'2024-02-16T11:45:43.413' AS DateTime), N'ArchivingInit', CAST(N'2024-02-16T11:45:43.413' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (19, N'ET9900067', N'CRIC.', 40, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (20, N'ET9900127', N'GTBS.', 4, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (21, N'ET9900049', N'FORX.', 0, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (22, N'ET9900106', N'REVY.', 1004, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (23, N'ET9900111', N'REES.', 65, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.123' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (24, N'ET9900038', N'RENS.', 8, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (25, N'ET9900128', N'LCLG.', 540, NULL, 0, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'clababidi', CAST(N'2025-02-17T15:33:28.807' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (26, N'ET9900066', N'BOMA.', 449, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (27, N'ET9900065', N'AMSS.', 1963, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (28, N'ET9900083', N'ARIC.', 1, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (29, N'ET9900093', N'LEGA.', 356, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (30, N'ET9900027', N'VIRE.', 2131, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (31, N'ET9900013', N'DFPE.', 1298, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (32, N'ET9900086', N'MARS.', 0, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (33, N'ET9900047', N'BRGCC.', 4, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (34, N'ET9900010', N'CONS.', 0, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (35, N'ET9900099', N'ITSS.', 15, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (36, N'ET9900004', N'REGL.', 58, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (37, N'ET9900000', N'PAIE.', 4, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (38, N'ET9900054', N'HRBP.', 2160, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (39, N'ET9900107', N'SHRI.', 234, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.140' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (40, N'ET9900016', N'CAPE.', 365, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.157' AS DateTime), N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.157' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (41, N'ET9900119', N'INSP.', 202, NULL, 1, N'AlternaSysUser', CAST(N'2024-11-26T09:12:29.157' AS DateTime), N'clababidi', CAST(N'2025-02-17T13:36:16.710' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (42, N'ET9900130', N'LC.', 2434, NULL, 1, N'AlternaSysUser', CAST(N'2025-02-17T15:11:15.177' AS DateTime), N'clababidi', CAST(N'2025-02-18T10:18:40.353' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (43, N'ET9900131', N'LG.', 540, NULL, 1, N'AlternaSysUser', CAST(N'2025-02-17T15:11:15.177' AS DateTime), N'AlternaSysUser', CAST(N'2025-02-17T15:11:15.177' AS DateTime))
INSERT [dbo].[t_Sequence] ([SequenceId], [Owner], [Prefix], [LastIndex], [Suffix], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (44, N'ET9900132', N'ET9900132.', 2, NULL, 0, N'AlternaSystem', CAST(N'2025-10-30T11:21:42.067' AS DateTime), N'AlternaSystem', CAST(N'2025-10-30T11:21:42.067' AS DateTime))
SET IDENTITY_INSERT [dbo].[t_Sequence] OFF
SET ANSI_PADDING ON

GO
/****** Object:  Index [IX_t_Sequence]    Script Date: 07/11/2025 11:23:22 AM ******/
ALTER TABLE [dbo].[t_Sequence] ADD  CONSTRAINT [IX_t_Sequence] UNIQUE NONCLUSTERED 
(
	[Owner] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, IGNORE_DUP_KEY = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
GO
ALTER TABLE [dbo].[Configuration] ADD  CONSTRAINT [DF__Configura__Creat__55009F39]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[Configuration] ADD  CONSTRAINT [DF__Configura__LastM__55F4C372]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[lkp_FileType] ADD  CONSTRAINT [DF__lkp_FileT__Creat__160F4887]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[lkp_FileType] ADD  CONSTRAINT [DF__lkp_FileT__LastM__17036CC0]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[lkp_Status] ADD  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[lkp_Status] ADD  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF_t_Container_isDeleted]  DEFAULT ((0)) FOR [isDeleted]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF_t_Container_isNotified]  DEFAULT ((0)) FOR [isNotified]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF__t_Contain__Creat__1BC821DD]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF__t_Contain__LastM__1CBC4616]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_ContainerStatus] ADD  CONSTRAINT [DF_t_ContainerStatus_isCurrentStatus]  DEFAULT ((1)) FOR [isCurrentStatus]
GO
ALTER TABLE [dbo].[t_ContainerStatus] ADD  CONSTRAINT [DF__t_Contain__Creat__25518C17]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_ContainerStatus] ADD  CONSTRAINT [DF__t_Contain__LastM__2645B050]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_CurrentContainerFileRelationship] ADD  CONSTRAINT [DF__t_Current__Creat__2739D489]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_CurrentContainerFileRelationship] ADD  CONSTRAINT [DF__t_Current__LastM__282DF8C2]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF_t_Customer_CreatedBy]  DEFAULT ('ETLSysUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF__t_Custome__Creat__29221CFB]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF_t_Customer_LastModifiedBy]  DEFAULT ('ETLSysUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF__t_Custome__LastM__2A164134]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF_t_File_FromDate]  DEFAULT (getdate()) FOR [FromDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF_t_File_ToDate]  DEFAULT (getdate()) FOR [ToDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF_t_File_isDeleted]  DEFAULT ((0)) FOR [isDeleted]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF__t_File__CreatedD__2B0A656D]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF__t_File__LastModi__2BFE89A6]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_FileStatus] ADD  CONSTRAINT [DF_t_FileStatus_isCurrentStatus]  DEFAULT ((1)) FOR [isCurrentStatus]
GO
ALTER TABLE [dbo].[t_FileStatus] ADD  CONSTRAINT [DF__t_FileSta__Creat__32AB8735]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_FileStatus] ADD  CONSTRAINT [DF__t_FileSta__LastM__339FAB6E]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_PDF] ADD  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_PDF] ADD  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Sequence] ADD  CONSTRAINT [DF_t_Sequence_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[t_Sequence] ADD  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Sequence] ADD  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
/****** Object:  StoredProcedure [dbo].[usp_CheckPDFExistsForContainer]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_CheckPDFExistsForContainer]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        CASE 
            WHEN EXISTS (
                SELECT 1 
                FROM t_PDF 
                WHERE Request LIKE '%"ContainerID": "' + @ContainerCode + '"%'
                    AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
            ) 
            THEN 1 
            ELSE 0 
        END AS PDFExists,
        CASE 
            WHEN EXISTS (
                SELECT 1 
                FROM t_PDF 
                WHERE Request LIKE '%"ContainerID": "' + @ContainerCode + '"%'
                    AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
                    AND PDF IS NOT NULL
                    AND DATALENGTH(PDF) > 0
            ) 
            THEN 1 
            ELSE 0 
        END AS PDFBinaryExists;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_EditContainerStatus]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
  -- =============================================
  -- Author:		<Pierre Abou Serhal>
  -- Create date: <2023/10/25>
  -- Description:	<Edit Container Status>
  -- =============================================
  CREATE PROCEDURE [dbo].[usp_EditContainerStatus] (
    -- PARAMETER LIST
    @ContainerId INT,
    @StatusCode NVARCHAR(10),
    @HoldingEntityCode NVARCHAR(max),
    @User NVARCHAR(250)
  ) AS BEGIN
SET
  NOCOUNT ON;
 DECLARE @FirstActiveBranch nvarchar(9);

SET
  @FirstActiveBranch = COALESCE(
    (
      SELECT
        TOP 1 Owner
      FROM
        t_Sequence
      WHERE
        IsActive = 1
        AND Owner IN (
          SELECT
            value
          FROM
            string_split(@HoldingEntityCode, ',')
        )
    ),
    'ERROR'
  );
  
UPDATE
  t_ContainerStatus
SET
  isCurrentStatus = 0
WHERE
  ContainerId = @ContainerId
UPDATE
  t_Container
SET
  StatusCode = @StatusCode,
  LastModifiedBy = @User,
  LastModifiedDate = GETDATE()
WHERE
  Id = @ContainerId
INSERT INTO
  t_ContainerStatus (
    ContainerId,
    StatusCode,
    HoldingEntityCode,
    isCurrentStatus,
    CreatedBy,
    LastModifiedBy
  )
VALUES
  (
    @ContainerId,
    @StatusCode,
    @FirstActiveBranch,
    1,
    @User,
    @User
  )
SELECT
  Id,
  CompanyCode,
  CurrentLocation,
  StatusCode,
  isDeleted
FROM
  t_Container
WHERE
  Id = @ContainerId
END



GO
/****** Object:  StoredProcedure [dbo].[usp_GetContainerDataForPDFGeneration]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Patricia Sammia>
-- Create date: <2025/10/30>
-- Description:	<Get Container Data For PDF Generation>
-- =============================================

CREATE PROCEDURE [dbo].[usp_GetContainerDataForPDFGeneration]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        c.Id AS ContainerId,
        c.Code AS ContainerCode,
        c.CompanyCode,
        c.Entity,
        c.ArchivingDate,
        c.StatusCode,
        f.Id AS FileId,
        f.Name AS FileName,
        f.CustomerId,
        f.FromDate,
        f.ToDate,
        f.AdditionalInfo,
        ft.ArchivingPeriod,
        ft.Description AS FileTypeDescription,
        CASE 
            WHEN f.CustomerId IS NOT NULL THEN 'CUSTOMER'
            WHEN c.CompanyCode LIKE 'LB%' THEN 'BRANCH'
            WHEN c.CompanyCode LIKE 'ET%' THEN 'ENTITY'
            ELSE 'UNKNOWN'
        END AS DocumentType
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    INNER JOIN lkp_FileType ft ON f.FileTypeCode = ft.Code
    WHERE c.Code = @ContainerCode 
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
    ORDER BY f.Name;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetContainerFilesForBackfill]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetContainerFilesForBackfill]
    @ContainerId INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        f.Id AS FileId,
        f.Name AS FileName,
        f.CustomerId,
        f.FromDate,
        f.ToDate,
        f.AdditionalInfo,
        f.CompanyCode,
        ft.ArchivingPeriod,
        ft.Description AS FileTypeDescription
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    INNER JOIN lkp_FileType ft ON f.FileTypeCode = ft.Code
    WHERE c.Id = @ContainerId
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
    ORDER BY f.Name;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetContainerSentByUser]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetContainerSentByUser]
    @ContainerId INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Get the user who changed status to SENT
    SELECT TOP 1 
        cs.CreatedBy AS SentByUser,
        cs.CreatedDate AS SentDate,
        cs.HoldingEntityCode
    FROM t_ContainerStatus cs
    WHERE cs.ContainerId = @ContainerId
        AND cs.StatusCode = 'SENT'
    ORDER BY cs.CreatedDate DESC;
    
    -- If no SENT status found, get the last user who modified the container
    IF @@ROWCOUNT = 0
    BEGIN
        SELECT 
            c.LastModifiedBy AS SentByUser,
            c.LastModifiedDate AS SentDate,
            c.Entity AS HoldingEntityCode
        FROM t_Container c
        WHERE c.Id = @ContainerId;
    END
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetContainerSentByUserByCode]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetContainerSentByUserByCode]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    -- Get the user who changed status to SENT based on container code
    SELECT TOP 1 
        cs.CreatedBy AS SentByUser,
        cs.CreatedDate AS SentDate,
        cs.HoldingEntityCode,
        c.Id AS ContainerId
    FROM t_Container c
    INNER JOIN t_ContainerStatus cs ON c.Id = cs.ContainerId
    WHERE c.Code = @ContainerCode
        AND cs.StatusCode = 'SENT'
    ORDER BY cs.CreatedDate DESC;
    
    -- If no SENT status found, get the last user who modified the container
    IF @@ROWCOUNT = 0
    BEGIN
        SELECT 
            c.LastModifiedBy AS SentByUser,
            c.LastModifiedDate AS SentDate,
            c.Entity AS HoldingEntityCode,
            c.Id AS ContainerId
        FROM t_Container c
        WHERE c.Code = @ContainerCode;
    END
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetContainersWithoutPDF]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetContainersWithoutPDF]
AS
BEGIN
    SET NOCOUNT ON;

    SELECT DISTINCT 
        c.Id,
        c.Code,
        c.CompanyCode,
        c.Entity,
        c.StatusCode,
        c.ArchivingDate,
        c.CreatedDate
    FROM t_Container c
    WHERE c.StatusCode IN ('SENT', 'RECEIVED', 'TOBEDESTR', 'DESTROYED')
        AND c.isDeleted = 0
        AND NOT EXISTS (
            SELECT 1 
            FROM t_PDF p 
            WHERE p.Request LIKE '%"ContainerID": "' + c.Code + '"%'
                AND p.ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
        )
    ORDER BY c.CreatedDate DESC;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetCustomerFilesByContainer]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetCustomerFilesByContainer]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        f.Name AS DocumentType,
        f.CustomerId,
        CAST(f.CustomerId AS NVARCHAR(50)) AS CustomerIdString
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    WHERE c.Code = @ContainerCode 
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
        AND f.CustomerId IS NOT NULL
    ORDER BY f.Name, f.CustomerId;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetEntityByCode]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetEntityByCode]
    @EntityCode NVARCHAR(11)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        Code,
        Description,
        HasFullAccess,
        Category
    FROM lkp_Entity
    WHERE Code = @EntityCode;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetPDFRequestByBoxReference]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetPDFRequestByBoxReference]
    @BoxReference NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TOP 1 Request, ApiMethod
    FROM t_PDF 
    WHERE Request LIKE '%"ContainerID": "' + @BoxReference + '"%' 
        AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
    ORDER BY CreatedDate DESC;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_GetPDFVarBinaryByBoxReference]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_GetPDFVarBinaryByBoxReference]
    @BoxReference NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TOP 1 PDF 
    FROM t_PDF 
    WHERE Request LIKE '%"ContainerID": "' + @BoxReference + '"%' 
        AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
        AND PDF IS NOT NULL
        AND DATALENGTH(PDF) > 0
    ORDER BY CreatedDate DESC;
END

GO
/****** Object:  StoredProcedure [dbo].[usp_Insert_Into_All_Tables]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Patricia Sammia>
-- Create date: <2025/10/30>
-- Description:	<Insert Data of Old Boxes into all tables due to excel file upload>
-- =============================================
CREATE PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250)
AS 
BEGIN 
    SET NOCOUNT ON;
	SELECT 1;  
    DECLARE @Now DATETIME = GETDATE(); 
	--TODO: to be updated
    DECLARE @FileTypeCode NVARCHAR(50) = '205';
    DECLARE @SystemUser NVARCHAR(250) = 'AlternaSystem'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],[Code],0,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Company] comp 
            WHERE comp.[Code] = input.[Code]
        );

        -- Temp tables to hold inserted IDs and link them back to input data 
        DECLARE @InsertedContainers TABLE( 
            RowId INT, 
            ContainerId INT 
        ); 

        DECLARE @InsertedFiles TABLE( 
            RowId INT, 
            FileId INT 
        ); 

        -- Insert Containers with unique Code + CompanyCode combination
        WITH UniqueContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate,
                   ROW_NUMBER() OVER (PARTITION BY BoxRef, Code ORDER BY RowId) as rn
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_Container] cont
                WHERE cont.[Code] = input.[BoxRef]
                AND cont.[CompanyCode] = input.[Code]
            )
        ),
        ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM UniqueContainerSource
            WHERE rn = 1  -- Only take the first occurrence of each unique BoxRef + CompanyCode combination
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0  -- Always insert, never match
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @SystemUser, @Now, @SystemUser, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Also capture existing container IDs for ALL rows (including duplicates within input)
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef]
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- For input rows with duplicate BoxRef + CompanyCode that weren't inserted, map them to the inserted container
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, ic.ContainerId
        FROM @P__Old_Boxes input
        INNER JOIN @InsertedContainers ic ON EXISTS (
            SELECT 1 FROM @P__Old_Boxes input2 
            WHERE input2.RowId = ic.RowId 
            AND input2.BoxRef = input.BoxRef
            AND input2.Code = input.Code
        )
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- Insert new File Type with auto-incrementing FileTypeCode (PK)
        -- DESCRIPTION + ENTITY MUST BE UNIQUE
        DECLARE @NewFileTypes TABLE (
            Description NVARCHAR(250),
            Entity NVARCHAR(50),
            ArchivingPeriod INT,
            NextCode INT
        );
        
        -- Get the current maximum FileTypeCode
        DECLARE @MaxFileTypeCode INT;
        SELECT @MaxFileTypeCode = ISNULL(MAX(CAST(Code AS INT)), 0) 
        FROM [dbo].[lkp_FileType] 
        WHERE ISNUMERIC(Code) = 1;
        
        -- Insert unique Description + Entity combinations that don't exist with incremented codes
        WITH UniqueNewFileTypes AS (
            SELECT DISTINCT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   input.[ArchivingPeriod]
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        -- Insert new File Type records with incremented FileTypeCode
        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               @SystemUser,
               @Now,
               @SystemUser,
               @Now
        FROM @NewFileTypes nft;

        -- Get the FileTypeCode for each file (either existing or newly created)
        DECLARE @FileTypeCodes TABLE (
            RowId INT,
            FileTypeCode NVARCHAR(50)
        );
        
        INSERT INTO @FileTypeCodes (RowId, FileTypeCode)
        SELECT input.RowId, ft.Code
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.[Entity] = input.[Code] 
            AND ft.[Description] = input.[FileName];

        -- MODIFIED: Insert Files - Create separate file records for each container
        -- Each file gets its own ID even if FileName is duplicated across containers
        -- No uniqueness check - every row gets a new file record
        INSERT INTO [dbo].[t_File] 
        ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
        OUTPUT inserted.Id
        INTO @InsertedFiles(FileId)
        SELECT null, 
               input.FileName, 
               ftc.FileTypeCode, 
               'FINAL', 
               input.Code, 
               null, 
               null, 
               input.AdditionalInfo, 
               0, 
               @SystemUser, 
               @Now, 
               @SystemUser, 
               @Now
        FROM @P__Old_Boxes input
        INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
        ORDER BY input.RowId;

        -- Map the inserted FileIds back to RowIds in the correct order
        DECLARE @RowIdMapping TABLE (
            RowId INT,
            FileId INT,
            RowNum INT
        );

        INSERT INTO @RowIdMapping (RowId, RowNum)
        SELECT RowId, ROW_NUMBER() OVER (ORDER BY RowId) as RowNum
        FROM @P__Old_Boxes;

        WITH NumberedInsertedFiles AS (
            SELECT FileId, ROW_NUMBER() OVER (ORDER BY FileId) as RowNum
            FROM @InsertedFiles
        )
        UPDATE @InsertedFiles
        SET RowId = rm.RowId
        FROM @InsertedFiles if_target
        INNER JOIN NumberedInsertedFiles nif ON if_target.FileId = nif.FileId
        INNER JOIN @RowIdMapping rm ON nif.RowNum = rm.RowNum;

        -- MODIFIED: Insert new Container File Relationship 
        -- Create relationships between containers and their associated files based on input data
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @SystemUser, @Now, @SystemUser, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId  -- Match based on input row relationship
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_CurrentContainerFileRelationship] rel
            WHERE rel.[FileId] = f.FileId AND rel.[ContainerId] = c.ContainerId
        );

        -- Insert new Sequence only if Owner doesn't already exist
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Sequence] seq
            WHERE seq.[Owner] = input.[Code]
        )
        AND input.[Code] NOT IN (
            SELECT i2.[Code] 
            FROM @P__Old_Boxes i2 
            WHERE i2.RowId < input.RowId
        );

        -- MODIFIED: Insert Box Statuses history based on input StatusCode
        -- RECEIVED containers: 2 statuses (SENT inactive, RECEIVED active)
        -- DESTROYED containers: 3 statuses (SENT inactive, RECEIVED inactive, DESTROYED active)
        
        -- Insert SENT status for all containers (always inactive - isCurrentStatus = 0, HoldingEntityCode = 'WH')
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT c.ContainerId,
               'SENT',
               'WH',  -- Always 'WH' for SENT status
               0,     -- Always inactive (false) for SENT
               CASE WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser ELSE i.[BoxSentBy] END,
               i.[BoxSentDate],
               CASE WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser ELSE i.[BoxSentBy] END,
               i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'SENT'
        );

        -- Insert RECEIVED status for containers with StatusCode RECEIVED or DESTROYED
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT c.ContainerId,
               'RECEIVED',
               i.[BoxSentBy],  -- Use BoxSentBy value for RECEIVED status
               CASE WHEN i.[StatusCode] = 'RECEIVED' THEN 1 ELSE 0 END,  -- Active only if final status is RECEIVED
               @SystemUser,  -- Always use AlternaSystem for RECEIVED status
               DATEADD(MINUTE, 1, i.[BoxSentDate]), -- Add 1 minute to ensure chronological order
               @SystemUser,  -- Always use AlternaSystem for RECEIVED status
               DATEADD(MINUTE, 1, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] IN ('RECEIVED', 'DESTROYED')
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'RECEIVED'
        );

        -- Insert DESTROYED status for containers with StatusCode DESTROYED only
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT c.ContainerId,
               'DESTROYED',
               'WH',  -- Always 'WH' for DESTROYED status
               1,     -- Always active (true) for DESTROYED as it's the final status
               @SystemUser,  -- Always use AlternaSystem for DESTROYED status
               DATEADD(MINUTE, 2, i.[BoxSentDate]), -- Add 2 minutes to ensure it's the latest
               @SystemUser,  -- Always use AlternaSystem for DESTROYED status
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'DESTROYED'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'DESTROYED'
        );

        COMMIT TRANSACTION; 
    END TRY 
    BEGIN CATCH 
        ROLLBACK TRANSACTION; 
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(); 
        DECLARE @ErrSeverity INT = ERROR_SEVERITY(); 
        RAISERROR (@ErrMsg, @ErrSeverity, 1); 
    END CATCH 
END;
GO
/****** Object:  StoredProcedure [dbo].[usp_InsertPDF]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[usp_InsertPDF]
(
    @PDF VARBINARY(MAX),
    @Request NVARCHAR(MAX),
    @ApiMethod NVARCHAR(500),
    @BranchList NVARCHAR(MAX),
    @Entity NVARCHAR(10),
    @User NVARCHAR(250)
)
AS 
BEGIN
    SET NOCOUNT ON;

    -- Check if PDF already exists for this container
    DECLARE @ContainerID NVARCHAR(50);
    
    -- Extract ContainerID from Request JSON
    SET @ContainerID = JSON_VALUE(@Request, '$.ContainerID');
    
    IF @ContainerID IS NOT NULL
    BEGIN
        -- Check if record exists
        IF EXISTS (
            SELECT 1 
            FROM t_PDF 
            WHERE Request LIKE '%"ContainerID": "' + @ContainerID + '"%'
                AND ApiMethod = @ApiMethod
        )
        BEGIN
            -- Update existing record with new PDF if provided
            IF @PDF IS NOT NULL AND DATALENGTH(@PDF) > 0
            BEGIN
                UPDATE t_PDF
                SET PDF = @PDF,
                    LastModifiedBy = @User,
                    LastModifiedDate = GETDATE()
                WHERE Request LIKE '%"ContainerID": "' + @ContainerID + '"%'
                    AND ApiMethod = @ApiMethod;
            END
        END
        ELSE
        BEGIN
            -- Insert new record
            INSERT INTO t_PDF (
                PDF,
                Request,
                ApiMethod,
                BranchList,
                Entity,
                CreatedBy,
                LastModifiedBy
            )
            VALUES (
                @PDF,
                @Request,
                @ApiMethod,
                @BranchList,
                @Entity,
                @User,
                @User
            );
        END
    END
    ELSE
    BEGIN
        -- If no ContainerID found, just insert
        INSERT INTO t_PDF (
            PDF,
            Request,
            ApiMethod,
            BranchList,
            Entity,
            CreatedBy,
            LastModifiedBy
        )
        VALUES (
            @PDF,
            @Request,
            @ApiMethod,
            @BranchList,
            @Entity,
            @User,
            @User
        );
    END
END

GO
/****** Object:  StoredProcedure [dbo].[usp_UpdateSequence]    Script Date: 07/11/2025 11:23:22 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
  -- =============================================
  -- Author:	  <Pierre Abou Serhal>
  -- Create date: <2023/10/23>
  -- Description: <Create Or Update Procedure For Table 't_Sequence'>
  -- =============================================
  CREATE PROCEDURE [dbo].[usp_UpdateSequence] (
    -- PARAMETERS LIST
    @SequenceId INT,
    @Owner NVARCHAR(50),
    @Prefix NVARCHAR(50) = NULL,
    @LastIndex BIGINT,
    @Suffix NVARCHAR(50) = NULL,
    @IsActive BIT,
	@User NVARCHAR(250)
  ) AS BEGIN
SET
  NOCOUNT ON;

-- CREATE
IF @SequenceId = -1
BEGIN
	INSERT INTO t_Sequence 
	(
		Owner,
		Prefix,
		LastIndex,
		Suffix,
		IsActive,
		CreatedBy,
		LastModifiedBy
	)
	VALUES
	(
		@Owner,
		@Prefix,
		@LastIndex,
		@Suffix,
		@IsActive,
		@User,
		@User
	)

	SET @SequenceId = SCOPE_IDENTITY()

END -- UPDATE
ELSE 
BEGIN
	UPDATE
		t_Sequence
	SET
		Owner = @Owner,
		Prefix = @Prefix,
		LastIndex = @LastIndex,
		Suffix = @Suffix,
		IsActive = @IsActive,
		LastModifiedBy = @User,
		LastModifiedDate = GETDATE()
	WHERE
		t_Sequence.SequenceId = @SequenceId
END

SELECT
	SequenceId,
	Owner,
	Prefix,
	LastIndex,
	Suffix,
	IsActive
FROM
	t_Sequence
WHERE
	t_Sequence.SequenceId = @SequenceId

END

GO
