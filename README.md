        [HttpPost]
        [Route("GetWarehouseContainers")]
        public GetWarehouseContainersRes GetWarehouseContainers(GetWarehouseContainersReq req)
        {
            GetWarehouseContainersRes response = new()
            {
                Req = req
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = req.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetWarehouseContainers",
                UserName = req.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(req.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : req.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(req.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : req.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(req.BaseReq.CurrentEntity) && String.IsNullOrEmpty(req.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(req.BaseReq.CurrentEntity)} and {nameof(req.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(req.BaseReq.CurrentEntity) ? String.Empty : req.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(req.BaseReq.CurrentBranch) ? String.Empty : req.BaseReq.CurrentBranch;

                LogInfo("GetWarehouseContainers Has been called with the following Request", correlationInfo);
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

                    LogInfo("Start of GetWarehouseContainers call", correlationInfo);

                    response.Resp = oBLL.GetWarehouseContainers(req);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Container have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetWarehouseContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetWarehouseContainers is completed", correlationInfo);
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
                response.Resp = [];

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
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = req.BaseReq.CorrelationId!;
                response.WebResp.User = req.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = req.BaseReq.CorrelationId!;
                response.WebResp.User = req.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }


        Severity	Code	Description	Project	File	Line	Suppression State
Error	CS1503	Argument 1: cannot convert from 'ALTERNA.ARCHIVING.BLL.GetWarehouseContainersReq' to 'ALTERNA.ARCHIVING.BLL.ExportWarehouseContainersReq'	ALTERNA.ARCHIVING.API	D:\@Workspace\deve-repo\alterna-archive\002 - APP\ALTERNA.ARCHIVING\ALTERNA.ARCHIVING.API\Controllers\ArchivingController.cs	7317	Active
How to fix it, based on the previous 
