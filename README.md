        #region UpdateFileImportDetails

        public async Task UpdateFileImportDetails(UpdateFileImportDetailsRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("@P__Id", request.Id);
            parameters.Add("@P__StatusCode", request.StatusCode);
            parameters.Add("@P__T24FileCheckSum", request.T24FileCheckSum);
            parameters.Add("@P__User", request.T24FileCheckSum);
            parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            _ = await dal.ExecuteQueryAsync<dynamic>(
                "usp_Update_File_Import_Details",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");
            if (!string.IsNullOrEmpty(storedProcedureErrorMessage))
            {
                throw new SGBLBadRequestException(storedProcedureErrorMessage);
            }
        }

        #endregion

        #region GenerateAndUploadT24FileViaFtp

        public async Task<GenerateAndUploadT24FileViaFtpResponse> GenerateAndUploadT24FileViaFtp(
            GenerateAndUploadT24FileViaFtpRequest request)
        {
            GenerateAndUploadT24FileViaFtpResponse response = new GenerateAndUploadT24FileViaFtpResponse()
            {
                Req = request,
            };

            GetFileImportContentByFileImportIdRequest getFileImportContentByFileImportIdRequest = new()
            {
                BaseReq = request.BaseReq,
                FileImportId = request.FileImportId
            };

            List<AlfaFileImportContent> fileImportContent =
                await GetFileImportContentByFileImportId(getFileImportContentByFileImportIdRequest);

            if (fileImportContent.Count == 0)
            {
                throw new SGBLBadRequestException("Cannot Generate T24 File For a File That Does Not Have Content!");
            }

            String fileName = $"{Utils.GetJulianDate()}-{Utils.GetPaddedSeconds()}_ALFA.txt";
            ;
            // String totalFileName = $"total_alfa_TSAL.txt";

            // Mapping FileImportContent Records to T24 File Fields
            List<Dictionary<String, String>> bodyRecords = GetAlfaT24FileContentRecords(fileImportContent);

            // Generate checksum for data before generating the t24 file
            string t24Checksum = Utils.GetObjectChecksum(bodyRecords);

            // Calling the Data Export API to Generate the actual T24 file
            String dataExportGenerateFileContentUrl = $"{_globalSettings.DataExportUrl}/GenerateAndUploadFileViaFtp";

            DataExportGenerateAndUploadFileViaFtpRequest dataExportGenerateFileContentRequest = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                FileCode = _globalSettings.GenerateT24FileCode,
                FileName = fileName,
                FilePath = _globalSettings.FtpConfigurations.AlfaFtpConfig.RemotePath,
                BodyRecords = bodyRecords
            };

            DataExportGenerateFileContentResponse responseData =
                await PostAsync<DataExportGenerateFileContentResponse>(dataExportGenerateFileContentRequest,
                    dataExportGenerateFileContentUrl);

            if (responseData is not { WebResp.StatusCode: HttpStatusCode.OK })
            {
                throw new SGBLBadRequestException(String.Join(',', responseData.WebResp.Errors));
            }

            // Update file import status and t24 checksum
            UpdateFileImportDetailsRequest updateFileImportDetailsRequest = new UpdateFileImportDetailsRequest()
            {
                Id = request.FileImportId,
                BaseReq = request.BaseReq,
                StatusCode = FileImportStatus.AwaitingT24FileReturned.ToString(),
                T24FileCheckSum = t24Checksum
            };

            await UpdateFileImportDetails(updateFileImportDetailsRequest);

            return response;
        }

        #endregion

        #region UploadAndValidateT24File

        public async Task<UploadAndValidateT24FileResponse> UploadAndValidateT24File(
            UploadAndValidateT24FileRequest request)
        {
            UploadAndValidateT24FileResponse response = new UploadAndValidateT24FileResponse()
            {
                Req = request
            };

            string dataExportValidateFileUrl = $"{_globalSettings.DataExportUrl}/ValidateFile";

            if (request.T24File == null || request.T24File.Length == 0)
            {
                throw new SGBLBadRequestException("Invalid t24 file");
            }

            //get the file configurations
            Common.Model.File fileConfig = await GetFileConfigurationByCode(nameof(FileConfigCodes.AlfaT24Config));

            Common.Dto.FileInfo dataExportFileInfo = fileConfig.ToFileInfoDto();

            // Read data from TEXT using data export
            byte[] fileBytes = await Utils.ToByteArrayAsync(request.T24File);

            string hex = Utils.ByteArrayToHexString(fileBytes);

            dataExportFileInfo.FileBinary = hex;

            DataExportValidateFileRequest dataExportValidateFileRequest = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                FileInfo = dataExportFileInfo,
                FileCode = _globalSettings.GenerateT24FileCode,
                FieldNames = fileConfig.FileColumns!.Select(x => x.ColTypeCode).ToList()
            };

            DataExportValidateFileResponse responseData = await PostAsync<DataExportValidateFileResponse>(dataExportValidateFileRequest, dataExportValidateFileUrl);

            if (responseData is not { WebResp.StatusCode: HttpStatusCode.OK })
            {
                throw new SGBLBadGateWayException(string.Join(Environment.NewLine, responseData.WebResp.Errors));
            }

            GetAlfaT24ClientsFromParsedDataResponse t24ParsedData = GetAlfaT24ClientsFromParsedData(responseData.ParsedDataList, fileConfig);

            // Generate checksum for data before generating the t24 file
            string t24Checksum = Utils.GetObjectChecksum(t24ParsedData.t24NormalizedData);

            return response;
        }

        #endregion
