== TelecomController.cs
        #region ProcessAlfaAttachment

        [HttpPost]
        [Route("ProcessAlfaAttachment")]
        public async Task<ProcessAlfaAttachmentResponse> ProcessAlfaAttachment([FromForm] ProcessAlfaAttachmentRequest request)
        {
            ProcessAlfaAttachmentResponse response = new ProcessAlfaAttachmentResponse()
            {
                Req = request,
                BaseResp = new BaseResp()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ProcessAlfaAttachment",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "ProcessAlfaAttachment has been called with the following Request";

                LogInfoJson(request, correlationInfo);

                await _bal.ProcessAlfaAttachment(request);

                correlationInfo.Reserved = "ProcessAlfaAttachment requested with the following response";

                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion
		==========

Bal.cs
======
        public async Task ProcessAlfaAttachment(ProcessAlfaAttachmentRequest request)
        {
            string dataExportValidateFileUrl = $"{_globalSettings.DataExportUrl}/ValidateFile";
            string archiveDate = $"{DateTime.Today:yyyyMMdd}";
            string archivePath = Path.Combine(_globalSettings.ArchivePath, archiveDate);

            if (request.Attachment == null || request.Attachment.Length == 0)
            {
                throw new SGBLBadRequestException("Invalid attachment");
            }

            //get the file configurations
            Common.Model.File fileConfig = await GetFileConfigurationByCode(nameof(FileConfigCodes.AlfaConfig));

            //
            Common.Dto.FileInfo dataExportFileInfo = fileConfig.ToFileInfoDto();

            //calculate checksum
            string checkSum = await Utils.GetFileChecksumAsync(request.Attachment);

            //check if file checksum already exists
            await CheckFileCheckSum(checkSum);

            //archive the imported file
            Utils.UploadFile(archivePath, request.Attachment);

            //read data from excel using data export
            byte[] fileBytes = await Utils.ToByteArrayAsync(request.Attachment);

            string hex = Utils.ByteArrayToHexString(fileBytes);

            dataExportFileInfo.FileBinary = hex;

            DataExportValidateFileRequest dataExportValidateFileRequest = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                FileInfo = dataExportFileInfo
            };

            DataExportValidateFileResponse responseData = await PostAsync<DataExportValidateFileResponse>(dataExportValidateFileRequest,dataExportValidateFileUrl);

            if (responseData is not { WebResp.StatusCode: HttpStatusCode.OK })
            {
                throw new SGBLBadGateWayException(string.Join(Environment.NewLine, responseData.WebResp.Errors));
            }

            List<AlfaClient> alfaClients = GetAlfaClientsFromParsedData(responseData.ParsedDataList, fileConfig!);

            //insert alfa clients to database
            await InsertAlfaClients(
                fileConfig.Id,
                request.CurrencyCode,
                request.Attachment.FileName,
                checkSum,
                request.Cycle,
                archiveDate,
                alfaClients,
                request.BaseReq.UserName);
        }
		=================

		CustomCode.cs
		==========
		        public async Task<Common.Model.File> GetFileConfigurationByCode(string fileCode)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters paramsOne = new DynamicParameters();
            paramsOne.Add("P__Code", fileCode);

            Common.Model.File? fileConfig = await dal.QueryMultipleAsync(
                "usp_Get_File_By_Code",
                paramsOne,
                CommandType.StoredProcedure,
                async multi =>
                {
                    Common.Model.File? file = multi.ReadFirstOrDefault<Common.Model.File>();

                    if (file != null)
                    {
                        IEnumerable<FileColumn> fileColumns = await multi.ReadAsync<FileColumn>();

                        file.FileColumns = fileColumns.ToList();
                    }

                    return file;
                });

            if (fileConfig == null)
            {
                throw new SGBLBadRequestException("file config code did not match any configurations in the system");
            }

            if (fileConfig is { FileColumns: null } || fileConfig.FileColumns.Count == 0)
            {
                throw new SGBLBadRequestException("file configuration is missing the columns configuration");
            }

            List<int> columnIdList = fileConfig.FileColumns.Select(c => c.Id).ToList();

            if (columnIdList.Count > 0)
            {
                string joinedColumns = string.Join(',', columnIdList);

                DynamicParameters paramsTwo = new DynamicParameters();
                paramsTwo.Add("P__ColumnIds", joinedColumns);

                IEnumerable<FileColumnValidation> fileColumnValidations =
                    await dal.ExecuteQueryAsync<FileColumnValidation>(
                        "usp_Get_File_Columns_Validation_By_File_Column_List",
                        paramsTwo,
                        CommandType.StoredProcedure,
                        DapperDal.CommandDirection.Select);

                foreach (FileColumn fileColumn in fileConfig.FileColumns)
                {
                    fileColumn.FileColumnValidations =
                        fileColumnValidations.Where(v => v.FileColumnId == fileColumn.Id).ToList();
                }
            }

            return fileConfig;
        }
		======================

		Mapper.cs
		========
		        public static Dto.FileInfo ToFileInfoDto(this Model.File model)
        {
            if (model == null) throw new ArgumentNullException(nameof(model));

            Dto.FileInfo fileInfo = new Dto.FileInfo()
            {
                FileType = (FileType)Int32.Parse(model.FileTypeCode),
                Delimiter = model.Delimeter ?? String.Empty,
                Encoding = (model.EncodingCode is not null) ? (EncodingEnum)Int32.Parse(model.EncodingCode) : EncodingEnum.UTF8, //Default,
                ValidateHeader = false,
                HeaderRowNumber = model.HeaderRow ?? 0,
                ValidateHeaderContainsFields = model.ValidateHeaderContainsFields?.Split(',').ToList() ?? [],
                ExcelPassword = model.Password
            };

            // Columns
            foreach (Model.FileColumn fileColumn in model.FileColumns ?? [])
            {
                List<CustomDataValidation> customDataValidations = [];

                foreach (Model.FileColumnValidation fileColumnValidation in fileColumn.FileColumnValidations ?? [])
                {
                    customDataValidations.Add(new()
                    {
                        InjestFilePropertyId = fileColumnValidation.IngestPropId,
                    });
                }

                fileInfo.Columns.Add(new()
                {
                    ColumnNumber = fileColumn.Column ?? 0,
                    FindFromHeader = fileColumn.FindFromHeader,
                    HeaderColumnName = fileColumn.HeaderColumnName ?? String.Empty,
                    StartRowNumber = fileColumn.StartRow,
                    ReadEndCondition = (ReadEndCondition)Int32.Parse(fileColumn.ReadEndConditionCode),
                    ReadEndConditionValue = fileColumn.ReadEndConditionValue ?? String.Empty,
                    CustomDataValidation = customDataValidations,
                });
            }

            return fileInfo;
        }
		=================

		CustomCode.cs
		========
		        private List<AlfaClient> GetAlfaClientsFromParsedData(List<ParsedData> parsedDataList,
            Common.Model.File fileConfig)
        {
            List<string> errors = new List<string>();
            List<AlfaClient> alfaClients = new List<AlfaClient>();

            // Resolve ParsedData per column (by index first, then header name)
            ParsedData? ResolveParsedData(FileColumn column) =>
                parsedDataList.FirstOrDefault(pd => pd.ColumnIndex == column.Column) ?? parsedDataList.FirstOrDefault(
                    pd =>
                        pd.ColumnName.Equals(column.HeaderColumnName, StringComparison.OrdinalIgnoreCase));

            // Resolve FileColumns from configuration
            FileColumn bankCodeColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Code));

            FileColumn bankNameColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Name));

            FileColumn bankBranchColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Branch));

            FileColumn bankAccountNumberColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Account_Number));

            FileColumn customerNameColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Customer_Name));

            FileColumn primaryAccountNumberColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Primary_Account_Number));

            FileColumn msisdnColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.MSISDN_Primary_Contact));

            FileColumn accountBalanceColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Account_Balance));

            FileColumn invoiceDateColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Invoice_Date));

            FileColumn amountPaidColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Amount_Paid));

            FileColumn sayrafaRateColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Sayrafa_Rate));

            // Resolve ParsedData
            ParsedData? bankCodeParsedData = ResolveParsedData(bankCodeColumn);
            ParsedData? bankNameParsedData = ResolveParsedData(bankNameColumn);
            ParsedData? bankBranchParsedData = ResolveParsedData(bankBranchColumn);
            ParsedData? bankAccountParsedData = ResolveParsedData(bankAccountNumberColumn);
            ParsedData? customerNameParsedData = ResolveParsedData(customerNameColumn);
            ParsedData? primaryAccountParsedData = ResolveParsedData(primaryAccountNumberColumn);
            ParsedData? msisdnParsedData = ResolveParsedData(msisdnColumn);
            ParsedData? balanceParsedData = ResolveParsedData(accountBalanceColumn);
            ParsedData? invoiceDateParsedData = ResolveParsedData(invoiceDateColumn);
            ParsedData? amountPaidParsedData = ResolveParsedData(amountPaidColumn);
            ParsedData? sayrafaRateParsedData = ResolveParsedData(sayrafaRateColumn);

            if (msisdnParsedData == null)
            {
                throw new Exception("MSISDN column not found in parsed data.");
            }

            if (balanceParsedData == null)
            {
                throw new Exception("Account Balance column not found in parsed data.");
            }

            // Build row-based dictionaries
            Dictionary<int, string?> bankCodeByRow =
                bankCodeParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> bankNameByRow =
                bankNameParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> bankBranchByRow =
                bankBranchParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> bankAccountByRow =
                bankAccountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> customerNameByRow =
                customerNameParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> primaryAccountByRow =
                primaryAccountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> msisdnByRow =
                msisdnParsedData.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim());

            Dictionary<int, string?> balanceByRow =
                balanceParsedData.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim());

            Dictionary<int, string?> invoiceDateByRow =
                invoiceDateParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> amountPaidByRow =
                amountPaidParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> sayrafaRateByRow =
                sayrafaRateParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            // Union of all row numbers
            List<int> rowNumbers =
                msisdnByRow.Keys
                    .Union(balanceByRow.Keys)
                    .OrderBy(r => r)
                    .ToList();

            foreach (int rowNumber in rowNumbers)
            {
                msisdnByRow.TryGetValue(rowNumber, out string? msisdn);
                balanceByRow.TryGetValue(rowNumber, out string? balanceRaw);

                if (string.IsNullOrWhiteSpace(msisdn))
                {
                    throw new SGBLBadRequestException($"Row {rowNumber}: MSISDN is empty.");
                }

                if (!decimal.TryParse(balanceRaw, out decimal balance))
                {
                    throw new SGBLBadRequestException($"Row {rowNumber}: Invalid Account Balance '{balanceRaw}'.");
                }

                string? invoiceDateRaw = Utils.GetT24DateFormat(invoiceDateByRow.GetValueOrDefault(rowNumber)!);

                if (!DateTime.TryParseExact(
                        invoiceDateRaw,
                        "yyyyMMdd",
                        CultureInfo.InvariantCulture,
                        DateTimeStyles.None,
                        out DateTime invoiceDate))
                {
                    throw new SGBLBadRequestException(
                        $"Row {rowNumber}: Invalid Invoice Date '{invoiceDateRaw}'. Expected format MM/dd/yyyy.");
                }

                string? amountPaidRaw = amountPaidByRow.GetValueOrDefault(rowNumber);
                decimal amountPaid = string.IsNullOrWhiteSpace(amountPaidRaw)
                    ? 0
                    : decimal.Parse(amountPaidRaw, CultureInfo.InvariantCulture);

                string? sayrafaRateRaw = sayrafaRateByRow.GetValueOrDefault(rowNumber);
                decimal sayrafaRate = string.IsNullOrWhiteSpace(sayrafaRateRaw)
                    ? 0
                    : decimal.Parse(sayrafaRateRaw, CultureInfo.InvariantCulture);

                alfaClients.Add(new AlfaClient
                {
                    BankCode = int.Parse(bankCodeByRow.GetValueOrDefault(rowNumber) ?? "0"),
                    BankName = bankNameByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    BankBranch = bankBranchByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    BankAccountNumber = bankAccountByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    CustomerName = customerNameByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    PrimaryAccountNumber = primaryAccountByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    MsisdnPrimaryContact = msisdn,
                    AccountBalance = balance,
                    InvoiceDate = invoiceDate,
                    AmountPaid = amountPaid,
                    SayrafaRate = sayrafaRate
                });
            }

            if (errors.Any())
            {
                throw new Exception(string.Join(Environment.NewLine, errors));
            }

            return alfaClients;
        }

        #region InsertAlfaClients

        public async Task InsertAlfaClients(
            int fileConfigId,
            CurrencyType currencyCode,
            string attachmentName,
            string checkSum,
            DateTime cycle,
            string directory,
            List<AlfaClient> alfaClients,
            string userName)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__FileId", fileConfigId);
            parameters.Add("P__CurrencyCode", currencyCode);
            parameters.Add("P__Name", attachmentName);
            parameters.Add("P__CheckSum", checkSum);
            parameters.Add("P__Directory", directory);
            parameters.Add("P__FileImportContentAlfa",
                GetFileImportContentAlfaDt(alfaClients).AsTableValuedParameter());
            parameters.Add("P__Cycle", cycle);
            parameters.Add("P__User", userName);
            parameters.Add("P__Error", direction: ParameterDirection.Output, size: 4000);

            _ = await dal.ExecuteQueryAsync<dynamic>(
                "usp_Bulk_Insert_File_Import_Content_Alfa",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            string errorMessage = parameters.Get<string>("P__Error");

            if (!string.IsNullOrWhiteSpace(errorMessage))
            {
                throw new SGBLBadRequestException(errorMessage);
            }
        }

        #endregion
		====================








		
