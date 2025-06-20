        #region ExportWarehouseContainers
        public ExportWarehouseContainersRes ExportWarehouseContainers(ExportWarehouseContainersReq req)
        {
            ExportWarehouseContainersRes response = new()
            {
                Req = req
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = req.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "DownloadPDF_WarehouseReport",
                UserName = req.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(req.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : req.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(req.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : req.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(req.BaseReq.CurrentEntity) && String.IsNullOrEmpty(req.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(DownloadPDFReq.BaseReq.CurrentEntity)} and {nameof(DownloadPDFReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(req.BaseReq.CurrentEntity) ? String.Empty : req.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(req.BaseReq.CurrentBranch) ? String.Empty : req.BaseReq.CurrentBranch;

                LogInfo("DownloadPDF_WarehouseReport Has been called with the following Request", correlationInfo);
                LogInfoJson(req, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(req) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateConfiguration call", correlationInfo);

                    response.Resp = oBLL.ExportWarehouseContainers(req);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to get box reference");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("DownloadPDF_WarehouseReport Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the DownloadPDF_WarehouseReport is completed", correlationInfo);
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
                response.WebResp.ResponseMessage = ex.StackTrace;

                //this was added in case correlation Id was invalid(null or Empty)
                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                //this was added in case Username was invalid(null or Empty)
                correlationInfo.UserName = response.WebResp.User;

                //don't forget to change status code in case of exception
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (SGBLInternalServerException ex)
            {
                response.WebResp.CorrelationId = req.BaseReq.CorrelationId!;
                response.WebResp.User = req.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = req.BaseReq.CorrelationId!;
                response.WebResp.User = req.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion


        #region ExportWarehouseContainers
        public byte[] ExportWarehouseContainers(ExportWarehouseContainersReq req)
        {
            DAL.DAL iDAL = new();
            List<Container> RetList = [];

            //TODO:(step 1) first fetch the data according to the filter
            DynamicParameters param = new();

            param.Add("FromDate", req.FromDate);
            param.Add("ToDate", req.ToDate);
            param.Add("Code", req.Code);
            param.Add("CompanyCode", req.CompanyCode);
            param.Add("StatusCode", req.StatusCode);

            RetList = iDAL.ExecuteQuery<Container>("usp_GetWarehouseContainers", param, CommandType.StoredProcedure,
                CommandDirection.Select);

            //TODO: (step 2) export the data to pdf
            ExportWarehouseContainersViewModel vm = new ExportWarehouseContainersViewModel()
            {
                Req = new GenerateWarehouseContainersReq()
                {
                    User = req.BaseReq.CurrentUser!,
                    BranchList = req.BaseReq.CurrentBranch!,
                    Entity = req.BaseReq.CurrentEntity!
                },
                WarehouseContainersList = RetList
            };

            return GenerateWarehouseContainersReport(vm);
        }
        #endregion      

        i have this error betwwen these 2 parts of syntax, how to fix it?
        Severity	Code	Description	Project	File	Line	Suppression State
Error	CS0029	Cannot implicitly convert type 'byte[]' to 'System.Collections.Generic.List<ALTERNA.ARCHIVING.BLL.Container>'	ALTERNA.ARCHIVING.API	D:\@Workspace\deve-repo\alterna-archive\002 - APP\ALTERNA.ARCHIVING\ALTERNA.ARCHIVING.API\Controllers\ArchivingController.cs	9864	Active
