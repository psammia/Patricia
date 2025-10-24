        public class ExcelValidationService
        {
            private const long MaxFileSize = 10 * 1024 * 1024; // 10MB
            private const string ExpectedMimeType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";

            public ValidationResult ValidateExcelFile(IFormFile file, ExcelValidationConfig config)
            {
                var result = new ValidationResult();

                // FILE-LEVEL VALIDATIONS - BASIC FILE CHECKS
                if (!ValidateBasicFileChecks(file, result))
                    return result;

                // FILE-LEVEL VALIDATIONS - SECURITY CHECKS
                if (!ValidateSecurityChecks(file, result))
                    return result;

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
                if (!string.IsNullOrEmpty(file.ContentType) &&
                    file.ContentType != ExpectedMimeType)
                {
                    result.AddError("File", $"Invalid MIME type. Expected {ExpectedMimeType} but got {file.ContentType}");
                    return false;
                }

                return true;
            }

            #endregion

            #region FILE-LEVEL VALIDATIONS - SECURITY CHECKS

            private bool ValidateSecurityChecks(IFormFile file, ValidationResult result)
            {
                        // Scan for macros (reject if .xlsm or contains VBA code)
                        var extension = Path.GetExtension(file.FileName)?.ToLowerInvariant();
                            //if (extension == ".xlsm")
                            //{
                            //    result.AddError("Security", "Macro-enabled files (.xlsm) are not allowed");
                            //    return false;
                            //}

                            // Validate file signature (ZIP signature for xlsx)
                            using var stream = file.OpenReadStream();
                var signature = new byte[4];
                stream.Read(signature, 0, 4);
                stream.Position = 0;

                            // Check for ZIP signature: PK (50 4B)
                            //if (!(signature[0] == 0x50 && signature[1] == 0x4B))
                            //{
                            //    result.AddError("Security", "File signature does not match Excel format");
                            //    return false;
                            //}

                return true;
            }

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
                //if (header.MustBeUnique)
                //{
                //    if (!uniquenessTrackers[header.Name].Add(cellValue))
                //    {
                //        result.AddError(location, $"Duplicate value '{cellValue}' found. This field must be unique");
                //    }
                //}
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
                        MustBeUnique = true,
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
