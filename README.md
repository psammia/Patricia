using System.Net;
using System.Text;
using ALTERNA.ARCHIVING.BLL;

using Microsoft.AspNetCore.Mvc;

using Newtonsoft.Json;

using static ALTERNA.ARCHIVING.BLL.BLL;

using static NLog.NLogUtil;

namespace ALTERNA.ARCHIVING.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public partial class ArchivingController : ControllerBase
    {
        #region GetAllConfigurations Controller
        [HttpPost]
        [Route("GetAllConfigurations")]
        public GetAllConfigurationsRes GetAllConfigurations(GetAllConfigurationsReq getallConfigurationReq)
        {
            GetAllConfigurationsRes response = new()
            {
                Req = getallConfigurationReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getallConfigurationReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllConfigurations",
                UserName = getallConfigurationReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getallConfigurationReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getallConfigurationReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getallConfigurationReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getallConfigurationReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getallConfigurationReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getallConfigurationReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getallConfigurationReq.BaseReq.CurrentEntity)} and {nameof(getallConfigurationReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getallConfigurationReq.BaseReq.CurrentEntity) ? String.Empty : getallConfigurationReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getallConfigurationReq.BaseReq.CurrentBranch) ? String.Empty : getallConfigurationReq.BaseReq.CurrentBranch;

                LogInfo("GetAllConfigurations Has been called with the following Request", correlationInfo);
                LogInfoJson(getallConfigurationReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getallConfigurationReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);
                    LogInfo("Start of GetAllConfigurations call", correlationInfo);

                    response.Resp = oBLL.GetAllConfigurations(getallConfigurationReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException("No active configurations have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetAllConfigurations Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetAllConfigurations is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getallConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getallConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getallConfigurationReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getallConfigurationReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getallConfigurationReq.BaseReq.CorrelationId!;

                response.WebResp.User = getallConfigurationReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getallConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = getallConfigurationReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetctiveConfigurations Controller
        [HttpPost]
        [Route("GetActiveConfigurations")]
        public GetActiveConfigurationsRes GetActiveConfigurations(GetActiveConfigurationsReq getactiveConfigurationReq)
        {
            GetActiveConfigurationsRes response = new()
            {
                Req = getactiveConfigurationReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getactiveConfigurationReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetActiveConfigurations",
                UserName = getactiveConfigurationReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getactiveConfigurationReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getactiveConfigurationReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getactiveConfigurationReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getactiveConfigurationReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getactiveConfigurationReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getactiveConfigurationReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getactiveConfigurationReq.BaseReq.CurrentEntity)} and {nameof(getactiveConfigurationReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getactiveConfigurationReq.BaseReq.CurrentEntity) ? String.Empty : getactiveConfigurationReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getactiveConfigurationReq.BaseReq.CurrentBranch) ? String.Empty : getactiveConfigurationReq.BaseReq.CurrentBranch;

                correlationInfo.RDirection = RequestDirection.Request;

                LogInfo("GetActiveConfigurations Has been called with the following Request", correlationInfo);
                LogInfoJson(getactiveConfigurationReq, correlationInfo);
                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getactiveConfigurationReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);
                    LogInfo("Start of GetActiveConfigurations call", correlationInfo);

                    response.Resp = oBLL.GetActiveConfigurations(getactiveConfigurationReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException("No active configurations have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCustomer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getactiveConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getactiveConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getactiveConfigurationReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getactiveConfigurationReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                //this was added in case correlation Id was invalid(null or Empty)
                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                //this was added in case Username was invalid(null or Empty)
                correlationInfo.UserName = response.WebResp.User;

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;

                LogInfo(ex.StackTrace, correlationInfo);
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = getactiveConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = getactiveConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.StatusCode = HttpStatusCode.NoContent;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = getactiveConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = getactiveConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;

                LogError(ex.StackTrace, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetCompany Controller
        [HttpPost]
        [Route("GetCompany")]
        public GetCompanyRes GetCompany(GetCompanyReq getCompanyReq)
        {
            GetCompanyRes response = new()
            {
                Req = getCompanyReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getCompanyReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetCompany",
                UserName = getCompanyReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getCompanyReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getCompanyReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getCompanyReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getCompanyReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getCompanyReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getCompanyReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getCompanyReq.BaseReq.CurrentEntity)} and {nameof(getCompanyReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getCompanyReq.BaseReq.CurrentEntity) ? String.Empty : getCompanyReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getCompanyReq.BaseReq.CurrentBranch) ? String.Empty : getCompanyReq.BaseReq.CurrentBranch;

                LogInfo("GetCompany Has been called with the following Request", correlationInfo);
                LogInfoJson(getCompanyReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getCompanyReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_CODE, getCompanyReq.Codes}
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetCompany call", correlationInfo);

                    response.Resp = oBLL.GetCompany(getCompanyReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException("No company have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;
                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCompany Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCompany is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getCompanyReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getCompanyReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getCompanyReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getCompanyReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getCompanyReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCompanyReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getCompanyReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCompanyReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetAllCompanies Controller
        [HttpPost]
        [Route("GetAllCompanies")]
        public GetAllCompaniesRes GetAllCompanies(GetAllCompaniesReq getAllCompaniesReq)
        {
            GetAllCompaniesRes response = new()
            {
                Req = getAllCompaniesReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getAllCompaniesReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllCompanies",
                UserName = getAllCompaniesReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getAllCompaniesReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getAllCompaniesReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getAllCompaniesReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getAllCompaniesReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getAllCompaniesReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getAllCompaniesReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getAllCompaniesReq.BaseReq.CurrentEntity)} and {nameof(getAllCompaniesReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getAllCompaniesReq.BaseReq.CurrentEntity) ? String.Empty : getAllCompaniesReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getAllCompaniesReq.BaseReq.CurrentBranch) ? String.Empty : getAllCompaniesReq.BaseReq.CurrentBranch;

                LogInfo("GetAllCompanies Has been called with the following Request", correlationInfo);
                LogInfoJson(getAllCompaniesReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getAllCompaniesReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetAllCompanies call", correlationInfo);

                    response.Resp = oBLL.GetAllCompanies(getAllCompaniesReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException("No company have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetAllCompanies Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCuGetAllCompaniesstomer is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getAllCompaniesReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getAllCompaniesReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getAllCompaniesReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getAllCompaniesReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getAllCompaniesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getAllCompaniesReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getAllCompaniesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getAllCompaniesReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetConfiguration Controller
        [HttpPost]
        [Route("GetConfiguration")]
        public GetConfigurationRes GetConfiguration(GetConfigurationReq getConfigurationReq)
        {
            GetConfigurationRes response = new()
            {
                Req = getConfigurationReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getConfigurationReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetConfiguration",
                UserName = getConfigurationReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getConfigurationReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getConfigurationReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getConfigurationReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getConfigurationReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getConfigurationReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getConfigurationReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getConfigurationReq.BaseReq.CurrentEntity)} and {nameof(getConfigurationReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getConfigurationReq.BaseReq.CurrentEntity) ? String.Empty : getConfigurationReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getConfigurationReq.BaseReq.CurrentBranch) ? String.Empty : getConfigurationReq.BaseReq.CurrentBranch;

                LogInfo("GetConfiguration Has been called with the following Request", correlationInfo);
                LogInfoJson(getConfigurationReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getConfigurationReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetConfiguration call", correlationInfo);

                    response.Resp = oBLL.GetConfiguration(getConfigurationReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Configuration have been found in our sytems matching: {getConfigurationReq.SettingName}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetConfiguration Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetConfiguration is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getConfigurationReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getConfigurationReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = getConfigurationReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = getConfigurationReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainer Controller
        [HttpPost]
        [Route("GetContainer")]
        public GetContainerRes GetContainer(GetContainerReq getContainer)
        {
            GetContainerRes response = new()
            {
                Req = getContainer
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainer.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainer",
                UserName = getContainer.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainer.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainer.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainer.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainer.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainer.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainer.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainer.BaseReq.CurrentEntity)} and {nameof(getContainer.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainer.BaseReq.CurrentEntity) ? String.Empty : getContainer.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainer.BaseReq.CurrentBranch) ? String.Empty : getContainer.BaseReq.CurrentBranch;

                LogInfo("GetContainer Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainer, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainer) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getContainer.Ids },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainer call", correlationInfo);

                    response.Resp = oBLL.GetContainer(getContainer);

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

                    LogInfo("GetContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCGetContainerustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainer.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainer.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainer.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainer.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getContainer.BaseReq.CorrelationId!;
                response.WebResp.User = getContainer.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainer.BaseReq.CorrelationId!;
                response.WebResp.User = getContainer.BaseReq.CurrentUser!;
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
        #endregion

        #region GetBranchContainers Controller
        [HttpPost]
        [Route("GetBranchContainers")]
        public GetBranchContainerRes GetBranchContainers(GetBranchContainerReq getBranchContainerReq)
        {
            GetBranchContainerRes response = new()
            {
                Req = getBranchContainerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getBranchContainerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetBranchContainers",
                UserName = getBranchContainerReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getBranchContainerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getBranchContainerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getBranchContainerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getBranchContainerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getBranchContainerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getBranchContainerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getBranchContainerReq.BaseReq.CurrentEntity)} and {nameof(getBranchContainerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getBranchContainerReq.BaseReq.CurrentEntity) ? String.Empty : getBranchContainerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getBranchContainerReq.BaseReq.CurrentBranch) ? String.Empty : getBranchContainerReq.BaseReq.CurrentBranch;

                LogInfo("GetBranchContainers Has been called with the following Request", correlationInfo);
                LogInfoJson(getBranchContainerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Start of GetBranchContainer call", correlationInfo);

                    response.Resp = oBLL.GetBranchContainers(getBranchContainerReq);

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

                    LogInfo("GetBranchContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetBranchContainer is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getBranchContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getBranchContainerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getBranchContainerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getBranchContainerReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getBranchContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = getBranchContainerReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getBranchContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = getBranchContainerReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetEntityContainers Controller
        [HttpPost]
        [Route("GetEntityContainers")]
        public GetEntityContainersRes GetEntityContainers(GetEntityContainerReq getEntityContainersReq)
        {
            GetEntityContainersRes response = new()
            {
                Req = getEntityContainersReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getEntityContainersReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetEntityContainers",
                UserName = getEntityContainersReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getEntityContainersReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getEntityContainersReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getEntityContainersReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getEntityContainersReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getEntityContainersReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getEntityContainersReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getEntityContainersReq.BaseReq.CurrentEntity)} and {nameof(getEntityContainersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getEntityContainersReq.BaseReq.CurrentEntity) ? String.Empty : getEntityContainersReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getEntityContainersReq.BaseReq.CurrentBranch) ? String.Empty : getEntityContainersReq.BaseReq.CurrentBranch;

                LogInfo("GetEntityContainers Has been called with the following Request", correlationInfo);
                LogInfoJson(getEntityContainersReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Start of GetEntityContainers call", correlationInfo);

                    response.Resp = oBLL.GetEntityContainers(getEntityContainersReq);

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

                    LogInfo("GetEntityContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetEntityContainers is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getEntityContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getEntityContainersReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getEntityContainersReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getEntityContainersReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getEntityContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityContainersReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getEntityContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityContainersReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerByCode Controller
        [HttpPost]
        [Route("GetContainerByCode")]
        public GetContainerByCodeRes GetContainerByCode(GetContainerByCodeReq getContainerByCode)
        {
            GetContainerByCodeRes response = new()
            {
                Req = getContainerByCode
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerByCode.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerByCode",
                UserName = getContainerByCode.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerByCode.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerByCode.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerByCode.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerByCode.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerByCode.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerByCode.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerByCode.BaseReq.CurrentEntity)} and {nameof(getContainerByCode.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerByCode.BaseReq.CurrentEntity) ? String.Empty : getContainerByCode.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerByCode.BaseReq.CurrentBranch) ? String.Empty : getContainerByCode.BaseReq.CurrentBranch;

                LogInfo("GetContainerByCode Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerByCode, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerByCode) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerByCode call", correlationInfo);

                    response.Resp = oBLL.GetContainerByCode(getContainerByCode);

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

                    LogInfo("GetContainerByCode Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCuGetContainerByCodestomer is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerByCode.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerByCode.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerByCode.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerByCode.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getContainerByCode.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByCode.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerByCode.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByCode.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerByStatus Controller
        [HttpPost]
        [Route("GetContainerByStatus")]
        public GetContainerByStatusRes GetContainerByStatus(GetContainerByStatusReq getContainerByStatus)
        {
            GetContainerByStatusRes response = new()
            {
                Req = getContainerByStatus
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerByStatus.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerByStatus",
                UserName = getContainerByStatus.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerByStatus.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerByStatus.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerByStatus.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerByStatus.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerByStatus.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerByStatus.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerByStatus.BaseReq.CurrentEntity)} and {nameof(getContainerByStatus.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerByStatus.BaseReq.CurrentEntity) ? String.Empty : getContainerByStatus.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerByStatus.BaseReq.CurrentBranch) ? String.Empty : getContainerByStatus.BaseReq.CurrentBranch;

                LogInfo("GetContainerByStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerByStatus, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerByStatus) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerByStatus call", correlationInfo);

                    response.Resp = oBLL.GetContainerByStatus(getContainerByStatus);

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

                    LogInfo("GetContainerByStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerByStatus is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerByStatus.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerByStatus.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerByStatus.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerByStatus.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                //this was added in case correlation Id was invalid (null or Empty)
                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                //this was added in case Username was invalid (null or Empty)
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
                response.WebResp.CorrelationId = getContainerByStatus.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByStatus.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerByStatus.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByStatus.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerByEntityOrBranch Controller
        [HttpPost]
        [Route("GetContainerByEntityOrBranch")]
        public GetContainerByEntityOrBranchRes GetContainerByEntityOrBranch(GetContainerByEntityOrBranchReq getContainerByEntityOrBranchReq)
        {
            GetContainerByEntityOrBranchRes response = new()
            {
                Req = getContainerByEntityOrBranchReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerByEntityOrBranchReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerByEntityOrBranch",
                UserName = getContainerByEntityOrBranchReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerByEntityOrBranchReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerByEntityOrBranchReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerByEntityOrBranchReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerByEntityOrBranchReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerByEntityOrBranchReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerByEntityOrBranchReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerByEntityOrBranchReq.BaseReq.CurrentEntity)} and {nameof(getContainerByEntityOrBranchReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerByEntityOrBranchReq.BaseReq.CurrentEntity) ? String.Empty : getContainerByEntityOrBranchReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerByEntityOrBranchReq.BaseReq.CurrentBranch) ? String.Empty : getContainerByEntityOrBranchReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerByEntityOrBranch Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerByEntityOrBranchReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerByEntityOrBranchReq) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerByEntityOrBranch call", correlationInfo);

                    response.Resp = oBLL.GetContainerByEntityOrBranch(getContainerByEntityOrBranchReq);

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

                    LogInfo("GetContainerByEntityOrBranch Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerByEntityOrBranch is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerByEntityOrBranchReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerByEntityOrBranchReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerByEntityOrBranchReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerByEntityOrBranchReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getContainerByEntityOrBranchReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByEntityOrBranchReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerByEntityOrBranchReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByEntityOrBranchReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetCustomerFilesByCustomerId Controller
        [HttpPost]
        [Route("GetCustomerFilesByCustomerId")]
        public GetCustomerFilesByCustomerIdRes GetCustomerFilesByCustomerId(GetCustomerFilesByCustomerIdReq GetCustomerFilesByCustomerId)
        {
            GetCustomerFilesByCustomerIdRes response = new()
            {
                Req = GetCustomerFilesByCustomerId
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = GetCustomerFilesByCustomerId.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetCustomerFilesByCustomerId",
                UserName = GetCustomerFilesByCustomerId.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(GetCustomerFilesByCustomerId.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : GetCustomerFilesByCustomerId.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(GetCustomerFilesByCustomerId.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : GetCustomerFilesByCustomerId.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(GetCustomerFilesByCustomerId.BaseReq.CurrentEntity) && String.IsNullOrEmpty(GetCustomerFilesByCustomerId.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(GetCustomerFilesByCustomerId.BaseReq.CurrentEntity)} and {nameof(GetCustomerFilesByCustomerId.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(GetCustomerFilesByCustomerId.BaseReq.CurrentEntity) ? String.Empty : GetCustomerFilesByCustomerId.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(GetCustomerFilesByCustomerId.BaseReq.CurrentBranch) ? String.Empty : GetCustomerFilesByCustomerId.BaseReq.CurrentBranch;

                LogInfo("GetCustomerFilesByCustomerId Has been called with the following Request", correlationInfo);
                LogInfoJson(GetCustomerFilesByCustomerId, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(GetCustomerFilesByCustomerId) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetCustomerFilesByCustomerId call", correlationInfo);

                    response.Resp = oBLL.GetCustomerFilesByCustomerId(GetCustomerFilesByCustomerId);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCustomerFilesByCustomerId Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCustomerFilesByCustomerId is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : GetCustomerFilesByCustomerId.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : GetCustomerFilesByCustomerId.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : GetCustomerFilesByCustomerId.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : GetCustomerFilesByCustomerId.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = GetCustomerFilesByCustomerId.BaseReq.CorrelationId!;
                response.WebResp.User = GetCustomerFilesByCustomerId.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = GetCustomerFilesByCustomerId.BaseReq.CorrelationId!;
                response.WebResp.User = GetCustomerFilesByCustomerId.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerFiles Controller
        [HttpPost]
        [Route("GetContainerFiles")]
        public GetContainerFilesRes GetContainerFiles(GetContainerFilesReq getContainerFiles)
        {
            GetContainerFilesRes response = new()
            {
                Req = getContainerFiles
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerFiles.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerFiles",
                UserName = getContainerFiles.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerFiles.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerFiles.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerFiles.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerFiles.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerFiles.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerFiles.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerFiles.BaseReq.CurrentEntity)} and {nameof(getContainerFiles.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerFiles.BaseReq.CurrentEntity) ? String.Empty : getContainerFiles.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerFiles.BaseReq.CurrentBranch) ? String.Empty : getContainerFiles.BaseReq.CurrentBranch;

                LogInfo("GetContainerFiles Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerFiles, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerFiles) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, getContainerFiles.ContainerId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerFiles call", correlationInfo);

                    response.Resp = oBLL.GetContainerFiles(getContainerFiles);

                    if (response.Resp == null || response.Resp.Id == 0)
                    {
                        throw new SGBLNotFoundException($"No File have been found in our sytems for the container: {getContainerFiles.ContainerId}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetContainerFiles Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerFiles is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerFiles.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerFiles.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerFiles.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerFiles.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = getContainerFiles.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerFiles.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = getContainerFiles.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerFiles.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetGeneralFilesByFileType Controller
        [HttpPost]
        [Route("GetGeneralFilesByFileType")]
        public GetGeneralFilesByFileTypeRes GetGeneralFilesByFileType(GetGeneralFilesByFileTypeReq getGeneralFilesByFileType)
        {
            GetGeneralFilesByFileTypeRes response = new()
            {
                Req = getGeneralFilesByFileType
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getGeneralFilesByFileType.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetGeneralFilesByFileType",
                UserName = getGeneralFilesByFileType.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getGeneralFilesByFileType.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getGeneralFilesByFileType.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getGeneralFilesByFileType.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getGeneralFilesByFileType.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getGeneralFilesByFileType.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getGeneralFilesByFileType.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getGeneralFilesByFileType.BaseReq.CurrentEntity)} and {nameof(getGeneralFilesByFileType.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getGeneralFilesByFileType.BaseReq.CurrentEntity) ? String.Empty : getGeneralFilesByFileType.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getGeneralFilesByFileType.BaseReq.CurrentBranch) ? String.Empty : getGeneralFilesByFileType.BaseReq.CurrentBranch;

                LogInfo("GetGeneralFilesByFileType Has been called with the following Request", correlationInfo);
                LogInfoJson(getGeneralFilesByFileType, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getGeneralFilesByFileType) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetGeneralFilesByFileType call", correlationInfo);

                    response.Resp = oBLL.GetGeneralFilesByFileType(getGeneralFilesByFileType);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetGeneralFilesByFileType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetGeneralFilesByFileType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getGeneralFilesByFileType.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getGeneralFilesByFileType.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getGeneralFilesByFileType.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getGeneralFilesByFileType.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getGeneralFilesByFileType.BaseReq.CorrelationId!;
                response.WebResp.User = getGeneralFilesByFileType.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getGeneralFilesByFileType.BaseReq.CorrelationId!;
                response.WebResp.User = getGeneralFilesByFileType.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerByFileId Controller
        [HttpPost]
        [Route("GetContainerByFileId")]
        public GetContainerByFileIdRes GetContainerByFileId(GetContainerByFileIdReq getContainerByFileIdReq)
        {
            GetContainerByFileIdRes response = new()
            {
                Req = getContainerByFileIdReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerByFileIdReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerByFileId",
                UserName = getContainerByFileIdReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerByFileIdReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerByFileIdReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerByFileIdReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerByFileIdReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerByFileIdReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerByFileIdReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerByFileIdReq.BaseReq.CurrentEntity)} and {nameof(getContainerByFileIdReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerByFileIdReq.BaseReq.CurrentEntity) ? String.Empty : getContainerByFileIdReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerByFileIdReq.BaseReq.CurrentBranch) ? String.Empty : getContainerByFileIdReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerByFileId Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerByFileIdReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerByFileIdReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getContainerByFileIdReq.FileIds}
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerByFileId call", correlationInfo);

                    response.Resp = oBLL.GetContainerByFileId(getContainerByFileIdReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Container have been found in our sytems for File Id: {String.Join(",", getContainerByFileIdReq.FileIds)}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetContainerByFileId Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerByFileId is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerByFileIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerByFileIdReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerByFileIdReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerByFileIdReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getContainerByFileIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByFileIdReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerByFileIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerByFileIdReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerStatus Controller
        [HttpPost]
        [Route("GetContainerStatus")]
        public GetContainerStatusRes GetContainerStatus(GetContainerStatusReq getContainerStatusReq)
        {
            GetContainerStatusRes response = new()
            {
                Req = getContainerStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerStatus",
                UserName = getContainerStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerStatusReq.BaseReq.CurrentEntity)} and {nameof(getContainerStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerStatusReq.BaseReq.CurrentEntity) ? String.Empty : getContainerStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerStatusReq.BaseReq.CurrentBranch) ? String.Empty : getContainerStatusReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerStatusReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getContainerStatusReq.Ids}
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerStatus call", correlationInfo);

                    response.Resp = oBLL.GetContainerStatus(getContainerStatusReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Container Status have been found for the following Ids: {String.Join(",", getContainerStatusReq.Ids)}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetContainerStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerStatus is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerStatusReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerStatusReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerStatusReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerType Controller
        [HttpPost]
        [Route("GetContainerType")]
        public GetContainerTypeRes GetContainerType(GetContainerTypeReq getContainerTypeReq)
        {
            GetContainerTypeRes response = new()
            {
                Req = getContainerTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerType",
                UserName = getContainerTypeReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerTypeReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerTypeReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerTypeReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerTypeReq.BaseReq.CurrentEntity)} and {nameof(getContainerTypeReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerTypeReq.BaseReq.CurrentEntity) ? String.Empty : getContainerTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerTypeReq.BaseReq.CurrentBranch) ? String.Empty : getContainerTypeReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerType Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerTypeReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerType call", correlationInfo);

                    response.Resp = oBLL.GetContainerType(getContainerTypeReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Container Type have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetContainerType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerTypeReq.BaseReq.CurrentBranch!;
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
                LogErrorJson(response, correlationInfo, ex)
;

                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = getContainerTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerTypeReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerTypeReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetEntity Controller
        [HttpPost]
        [Route("GetEntity")]
        public GetEntityRes GetEntity(GetEntityReq getEntityReq)
        {
            GetEntityRes response = new()
            {
                Req = getEntityReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getEntityReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetEntity",
                UserName = getEntityReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getEntityReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getEntityReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getEntityReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getEntityReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getEntityReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getEntityReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getEntityReq.BaseReq.CurrentEntity)} and {nameof(getEntityReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getEntityReq.BaseReq.CurrentEntity) ? String.Empty : getEntityReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getEntityReq.BaseReq.CurrentBranch) ? String.Empty : getEntityReq.BaseReq.CurrentBranch;

                LogInfo("GetEntity Has been called with the following Request", correlationInfo);
                LogInfoJson(getEntityReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getEntityReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetEntity call", correlationInfo);

                    response.Resp = oBLL.GetEntity(getEntityReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Entity have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetEntity Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetEntity is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getEntityReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getEntityReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getEntityReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetCurrentContainerFileRelationship Controller
        [HttpPost]
        [Route("GetCurrentContainerFileRelationship")]
        public GetCurrentContainerFileRelationshipRes GetCurrentContainerFileRelationship(GetCurrentContainerFileRelationshipReq getCurrentContainerFileRelationshipReq)
        {
            GetCurrentContainerFileRelationshipRes response = new()
            {
                Req = getCurrentContainerFileRelationshipReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getCurrentContainerFileRelationshipReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetCurrentContainerFileRelationship",
                UserName = getCurrentContainerFileRelationshipReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getCurrentContainerFileRelationshipReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getCurrentContainerFileRelationshipReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getCurrentContainerFileRelationshipReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getCurrentContainerFileRelationshipReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity)} and {nameof(getCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity) ? String.Empty : getCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch) ? String.Empty : getCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch;

                LogInfo("GetCurrentContainerFileRelationship Has been called with the following Request", correlationInfo);
                LogInfoJson(getCurrentContainerFileRelationshipReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getCurrentContainerFileRelationshipReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getCurrentContainerFileRelationshipReq.Ids}
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetCurrentContainerFileRelationship call", correlationInfo);

                    response.Resp = oBLL.GetCurrentContainerFileRelationship(getCurrentContainerFileRelationshipReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Relationship have been found for the following Ids: {String.Join(",", getCurrentContainerFileRelationshipReq.Ids)}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCurrentContainerFileRelationship Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCurrentContainerFileRelationship is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getCurrentContainerFileRelationshipReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getCurrentContainerFileRelationshipReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getCurrentContainerFileRelationshipReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCurrentContainerFileRelationshipReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getCurrentContainerFileRelationshipReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCurrentContainerFileRelationshipReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetCurrentFileStatusByFileId Controller
        [HttpPost]
        [Route("GetCurrentFileStatusByFileId")]
        public GetCurrentFileStatusByFileIdRes GetCurrentFileStatusByFileId(GetCurrentFileStatusByFileIdReq getCurrentFileStatusByFileIdReq)
        {
            GetCurrentFileStatusByFileIdRes response = new()
            {
                Req = getCurrentFileStatusByFileIdReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getCurrentFileStatusByFileIdReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetCurrentFileStatusByFileId",
                UserName = getCurrentFileStatusByFileIdReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getCurrentFileStatusByFileIdReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getCurrentFileStatusByFileIdReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getCurrentFileStatusByFileIdReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getCurrentFileStatusByFileIdReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getCurrentFileStatusByFileIdReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getCurrentFileStatusByFileIdReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getCurrentFileStatusByFileIdReq.BaseReq.CurrentEntity)} and {nameof(getCurrentFileStatusByFileIdReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getCurrentFileStatusByFileIdReq.BaseReq.CurrentEntity) ? String.Empty : getCurrentFileStatusByFileIdReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getCurrentFileStatusByFileIdReq.BaseReq.CurrentBranch) ? String.Empty : getCurrentFileStatusByFileIdReq.BaseReq.CurrentBranch;

                LogInfo("GetCurrentFileStatusByFileId Has been called with the following Request", correlationInfo);
                LogInfoJson(getCurrentFileStatusByFileIdReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getCurrentFileStatusByFileIdReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, getCurrentFileStatusByFileIdReq.FileId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetCurrentFileStatusByFileId call", correlationInfo);

                    response.Resp = oBLL.GetCurrentFileStatusByFileId(getCurrentFileStatusByFileIdReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Status have been found for the following File Id: {getCurrentFileStatusByFileIdReq.FileId}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCurrentFileStatusByFileId Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCurrentFileStatusByFileId is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getCurrentFileStatusByFileIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getCurrentFileStatusByFileIdReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getCurrentFileStatusByFileIdReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getCurrentFileStatusByFileIdReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getCurrentFileStatusByFileIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCurrentFileStatusByFileIdReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getCurrentFileStatusByFileIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCurrentFileStatusByFileIdReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetCustomerByWhere Controller
        [HttpPost]
        [Route("GetCustomerByWhere")]
        public GetCustomerByWhereRes GetCustomerByWhere(GetCustomerByWhereReq getCustomerByWhereReq)
        {
            GetCustomerByWhereRes response = new()
            {
                Req = getCustomerByWhereReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getCustomerByWhereReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetCustomerByWhere",
                UserName = getCustomerByWhereReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getCustomerByWhereReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getCustomerByWhereReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getCustomerByWhereReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getCustomerByWhereReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getCustomerByWhereReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getCustomerByWhereReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getCustomerByWhereReq.BaseReq.CurrentEntity)} and {nameof(getCustomerByWhereReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getCustomerByWhereReq.BaseReq.CurrentEntity) ? String.Empty : getCustomerByWhereReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getCustomerByWhereReq.BaseReq.CurrentBranch) ? String.Empty : getCustomerByWhereReq.BaseReq.CurrentBranch;

                LogInfo("GetCustomerByWhere Has been called with the following Request", correlationInfo);
                LogInfoJson(getCustomerByWhereReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getCustomerByWhereReq) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetCustomerByWhere call", correlationInfo);

                    response.Resp = oBLL.GetCustomerByWhere(getCustomerByWhereReq);

                    if (response.Resp == null || response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Customers have been found");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCustomer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getCustomerByWhereReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getCustomerByWhereReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getCustomerByWhereReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getCustomerByWhereReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getCustomerByWhereReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCustomerByWhereReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getCustomerByWhereReq.BaseReq.CorrelationId!;
                response.WebResp.User = getCustomerByWhereReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetArchivedFile Controller
        [HttpPost]
        [Route("GetArchivedFile")]
        public GetArchivedFileRes GetArchivedFile(GetArchivedFileReq getArchivedFileReq)
        {
            GetArchivedFileRes response = new()
            {
                Req = getArchivedFileReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getArchivedFileReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetArchivedFile",
                UserName = getArchivedFileReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getArchivedFileReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getArchivedFileReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getArchivedFileReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getArchivedFileReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getArchivedFileReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getArchivedFileReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getArchivedFileReq.BaseReq.CurrentEntity)} and {nameof(getArchivedFileReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getArchivedFileReq.BaseReq.CurrentEntity) ? String.Empty : getArchivedFileReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getArchivedFileReq.BaseReq.CurrentBranch) ? String.Empty : getArchivedFileReq.BaseReq.CurrentBranch;

                LogInfo("GetArchivedFile Has been called with the following Request", correlationInfo);
                LogInfoJson(getArchivedFileReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getArchivedFileReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getArchivedFileReq.Ids }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetArchivedFile call", correlationInfo);

                    response.Resp = oBLL.GetArchivedFile(getArchivedFileReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File have been found for the ids: {String.Join(",", getArchivedFileReq.Ids)}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetArchivedFile Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetArchivedFile is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getArchivedFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getArchivedFileReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getArchivedFileReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getArchivedFileReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getArchivedFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = getArchivedFileReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getArchivedFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = getArchivedFileReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetFileName Controller
        [HttpPost]
        [Route("GetFileName")]
        public GetFileNameRes GetFileName(GetFileNameReq getFileNameReq)
        {
            GetFileNameRes response = new()
            {
                Req = getFileNameReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getFileNameReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetFileName",
                UserName = getFileNameReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getFileNameReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getFileNameReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getFileNameReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getFileNameReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getFileNameReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getFileNameReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getFileNameReq.BaseReq.CurrentEntity)} and {nameof(getFileNameReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getFileNameReq.BaseReq.CurrentEntity) ? String.Empty : getFileNameReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getFileNameReq.BaseReq.CurrentBranch) ? String.Empty : getFileNameReq.BaseReq.CurrentBranch;

                LogInfo("GetFileName Has been called with the following Request", correlationInfo);
                LogInfoJson(getFileNameReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getFileNameReq) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetFileName call", correlationInfo);

                    response.Resp = oBLL.GetFileName(getFileNameReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Name been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetFileName Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetFileName is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getFileNameReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getFileNameReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getFileNameReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getFileNameReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getFileNameReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFileNameReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getFileNameReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFileNameReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetFilesByCustomerId Controller
        [HttpPost]
        [Route("GetFilesByCustomerId")]
        public GetFilesByCustomerIdRes GetFilesByCustomerId(GetFilesByCustomerIdReq getFilesByCustomerIdReq)
        {
            GetFilesByCustomerIdRes response = new()
            {
                Req = getFilesByCustomerIdReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getFilesByCustomerIdReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetFilesByCustomerId",
                UserName = getFilesByCustomerIdReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getFilesByCustomerIdReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getFilesByCustomerIdReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getFilesByCustomerIdReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getFilesByCustomerIdReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getFilesByCustomerIdReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getFilesByCustomerIdReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getFilesByCustomerIdReq.BaseReq.CurrentEntity)} and {nameof(getFilesByCustomerIdReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getFilesByCustomerIdReq.BaseReq.CurrentEntity) ? String.Empty : getFilesByCustomerIdReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getFilesByCustomerIdReq.BaseReq.CurrentBranch) ? String.Empty : getFilesByCustomerIdReq.BaseReq.CurrentBranch;

                LogInfo("GetFilesByCustomerId Has been called with the following Request", correlationInfo);
                LogInfoJson(getFilesByCustomerIdReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getFilesByCustomerIdReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, getFilesByCustomerIdReq.CustomerId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetFilesByCustomerId call", correlationInfo);

                    response.Resp = oBLL.GetFilesByCustomerId(getFilesByCustomerIdReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Files have been found for Customer Id: {getFilesByCustomerIdReq.CustomerId}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetFilesByCustomerId Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetFilesByCustomerId is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getFilesByCustomerIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getFilesByCustomerIdReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getFilesByCustomerIdReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getFilesByCustomerIdReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getFilesByCustomerIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFilesByCustomerIdReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getFilesByCustomerIdReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFilesByCustomerIdReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetFileStatus Controller
        [HttpPost]
        [Route("GetFileStatus")]
        public GetFileStatusRes GetFileStatus(GetFileStatusReq getFileStatusReq)
        {
            GetFileStatusRes response = new()
            {
                Req = getFileStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getFileStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetFileStatus",
                UserName = getFileStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getFileStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getFileStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getFileStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getFileStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getFileStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getFileStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getFileStatusReq.BaseReq.CurrentEntity)} and {nameof(getFileStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getFileStatusReq.BaseReq.CurrentEntity) ? String.Empty : getFileStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getFileStatusReq.BaseReq.CurrentBranch) ? String.Empty : getFileStatusReq.BaseReq.CurrentBranch;

                LogInfo("GetFileStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(getFileStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getFileStatusReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getFileStatusReq.Ids }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetFileStatus call", correlationInfo);

                    response.Resp = oBLL.GetFileStatus(getFileStatusReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Status have been found for the Ids: {String.Join(",", getFileStatusReq.Ids)}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetFileStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetFileStatus is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getFileStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getFileStatusReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFileStatusReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFileStatusReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetFileType Controller
        [HttpPost]
        [Route("GetFileType")]
        public GetFileTypeRes GetFileType(GetFileTypeReq getFileTypeReq)
        {
            GetFileTypeRes response = new()
            {
                Req = getFileTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getFileTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetFileType",
                UserName = getFileTypeReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getFileTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getFileTypeReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getFileTypeReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getFileTypeReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getFileTypeReq.BaseReq.CurrentEntity)} and {nameof(getFileTypeReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : getFileTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : getFileTypeReq.BaseReq.CurrentBranch;

                LogInfo("GetFileType Has been called with the following Request", correlationInfo);
                LogInfoJson(getFileTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getFileTypeReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetFileType call", correlationInfo);

                    response.Resp = oBLL.GetFileType(getFileTypeReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Types have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetFileType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetFileType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getFileTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getFileTypeReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFileTypeReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = getFileTypeReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetAllFileType Controller
        [HttpPost]
        [Route("GetAllFileType")]
        public GetAllFileTypeRes GetAllFileType(GetAllFileTypeReq GetAllFileTypeReq)
        {
            GetAllFileTypeRes response = new()
            {
                Req = GetAllFileTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = GetAllFileTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllFileType",
                UserName = GetAllFileTypeReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : GetAllFileTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : GetAllFileTypeReq.BaseReq.CurrentUser;



                String CurrentEntity = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : GetAllFileTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : GetAllFileTypeReq.BaseReq.CurrentBranch;

                LogInfo("GetAllFileType Has been called with the following Request", correlationInfo);
                LogInfoJson(GetAllFileTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
            {
                { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(GetAllFileTypeReq) },
            };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetAllFileType call", correlationInfo);

                    response.Resp = oBLL.GetAllFileType(GetAllFileTypeReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Types have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetAllFileType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetAllFileType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : GetAllFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : GetAllFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : GetAllFileTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : GetAllFileTypeReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = GetAllFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = GetAllFileTypeReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = GetAllFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = GetAllFileTypeReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetStatus Controller
        [HttpPost]
        [Route("GetStatus")]
        public GetStatusRes GetStatus(GetStatusReq getStatusReq)
        {
            GetStatusRes response = new()
            {
                Req = getStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetStatus",
                UserName = getStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getStatusReq.BaseReq.CurrentEntity)} and {nameof(getStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getStatusReq.BaseReq.CurrentEntity) ? String.Empty : getStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getStatusReq.BaseReq.CurrentBranch) ? String.Empty : getStatusReq.BaseReq.CurrentBranch;

                LogInfo("GetStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(getStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getStatusReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetStatus call", correlationInfo);

                    response.Resp = oBLL.GetStatus(getStatusReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Status have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetStatus is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getStatusReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getStatusReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getStatusReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetUserInteraction Controller
        [HttpPost]
        [Route("GetUserInteraction")]
        public GetUserInteractionRes GetUserInteraction(GetUserInteractionReq getUserInteractionReq)
        {
            GetUserInteractionRes response = new()
            {
                Req = getUserInteractionReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getUserInteractionReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetUserInteraction",
                UserName = getUserInteractionReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getUserInteractionReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getUserInteractionReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getUserInteractionReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getUserInteractionReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getUserInteractionReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getUserInteractionReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getUserInteractionReq.BaseReq.CurrentEntity)} and {nameof(getUserInteractionReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getUserInteractionReq.BaseReq.CurrentEntity) ? String.Empty : getUserInteractionReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getUserInteractionReq.BaseReq.CurrentBranch) ? String.Empty : getUserInteractionReq.BaseReq.CurrentBranch;

                LogInfo("GetUserInteraction Has been called with the following Request", correlationInfo);
                LogInfoJson(getUserInteractionReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getUserInteractionReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, getUserInteractionReq.Ids }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetUserInteraction call", correlationInfo);

                    response.Resp = oBLL.GetUserInteraction(getUserInteractionReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No User Interactions have been found for Ids: {String.Join(",", getUserInteractionReq.Ids)}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    LogInfo("GetUserInteraction Has been called with the following Request", correlationInfo);
                    LogInfoJson(getUserInteractionReq, correlationInfo);

                    correlationInfo.RDirection = RequestDirection.Processing;
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getUserInteractionReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getUserInteractionReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getUserInteractionReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getUserInteractionReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getUserInteractionReq.BaseReq.CorrelationId!;
                response.WebResp.User = getUserInteractionReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getUserInteractionReq.BaseReq.CorrelationId!;
                response.WebResp.User = getUserInteractionReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetUsers Controller
        [HttpPost]
        [Route("GetUsers")]
        public GetUsersRes GetUsers(GetUsersReq getUsersReq)
        {
            GetUsersRes response = new()
            {
                Req = getUsersReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getUsersReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetUsers",
                UserName = getUsersReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getUsersReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getUsersReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getUsersReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getUsersReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getUsersReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getUsersReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getUsersReq.BaseReq.CurrentEntity)} and {nameof(getUsersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getUsersReq.BaseReq.CurrentEntity) ? String.Empty : getUsersReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getUsersReq.BaseReq.CurrentBranch) ? String.Empty : getUsersReq.BaseReq.CurrentBranch;

                LogInfo("GetUsers Has been called with the following Request", correlationInfo);
                LogInfoJson(getUsersReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getUsersReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetUsers call", correlationInfo);

                    response.Resp = oBLL.GetUsers(getUsersReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Users have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetUsers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetUsers is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getUsersReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getUsersReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getUsersReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getUsersReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getUsersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getUsersReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getUsersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getUsersReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetWarehouse Controller
        [HttpPost]
        [Route("GetWarehouse")]
        public GetWarehouseRes GetWarehouse(GetWarehouseReq getWarehouseReq)
        {
            GetWarehouseRes response = new()
            {
                Req = getWarehouseReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getWarehouseReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetWarehouse",
                UserName = getWarehouseReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getWarehouseReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getWarehouseReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getWarehouseReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getWarehouseReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getWarehouseReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getWarehouseReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getWarehouseReq.BaseReq.CurrentEntity)} and {nameof(getWarehouseReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getWarehouseReq.BaseReq.CurrentEntity) ? String.Empty : getWarehouseReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getWarehouseReq.BaseReq.CurrentBranch) ? String.Empty : getWarehouseReq.BaseReq.CurrentBranch;

                LogInfo("GetWarehouse Has been called with the following Request", correlationInfo);
                LogInfoJson(getWarehouseReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getWarehouseReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_CODE, getWarehouseReq.Codes }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetWarehouse call", correlationInfo);

                    response.Resp = oBLL.GetWarehouse(getWarehouseReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Warehouses have been found for the given codes: {getWarehouseReq.Codes}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetWarehouse Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetWarehouse is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getWarehouseReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getWarehouseReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = getWarehouseReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = getWarehouseReq.BaseReq.CurrentUser!;
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
        #endregion

        #region UpdateConfiguration Controller
        [HttpPost]
        [Route("UpdateConfiguration")]
        public UpdateConfigurationRes UpdateConfiguration(UpdateConfigurationReq updateConfigurationReq)
        {
            UpdateConfigurationRes response = new()
            {
                Req = updateConfigurationReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateConfigurationReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateConfiguration",
                UserName = updateConfigurationReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateConfigurationReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateConfigurationReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateConfigurationReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateConfigurationReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateConfigurationReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateConfigurationReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateConfigurationReq.BaseReq.CurrentEntity)} and {nameof(updateConfigurationReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateConfigurationReq.BaseReq.CurrentEntity) ? String.Empty : updateConfigurationReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateConfigurationReq.BaseReq.CurrentBranch) ? String.Empty : updateConfigurationReq.BaseReq.CurrentBranch;

                LogInfo("UpdateConfiguration Has been called with the following Request", correlationInfo);
                LogInfoJson(updateConfigurationReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateConfigurationReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateConfigurationReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateConfiguration call", correlationInfo);

                    response.Resp = oBLL.UpdateConfiguration(updateConfigurationReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Configuration");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateConfiguration Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateConfiguration is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateConfigurationReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateConfigurationReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateConfigurationReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateConfigurationReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region DownloadPDF Controller
        [HttpPost]
        [Route("DownloadPDF")]
        public DownloadPDFRes DownloadPDF(DownloadPDFReq downloadPDFReq)
        {
            DownloadPDFRes response = new()
            {
                Req = downloadPDFReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = downloadPDFReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "DownloadPDF",
                UserName = downloadPDFReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(downloadPDFReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : downloadPDFReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(downloadPDFReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : downloadPDFReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(downloadPDFReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(downloadPDFReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(DownloadPDFReq.BaseReq.CurrentEntity)} and {nameof(DownloadPDFReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(downloadPDFReq.BaseReq.CurrentEntity) ? String.Empty : downloadPDFReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(downloadPDFReq.BaseReq.CurrentBranch) ? String.Empty : downloadPDFReq.BaseReq.CurrentBranch;

                LogInfo("DownloadPDF Has been called with the following Request", correlationInfo);
                LogInfoJson(downloadPDFReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(downloadPDFReq) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateConfiguration call", correlationInfo);

                    response.Resp = oBLL.DownloadPDF(downloadPDFReq);

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

                    LogInfo("GetCustomer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : downloadPDFReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : downloadPDFReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : downloadPDFReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : downloadPDFReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = downloadPDFReq.BaseReq.CorrelationId!;
                response.WebResp.User = downloadPDFReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = downloadPDFReq.BaseReq.CorrelationId!;
                response.WebResp.User = downloadPDFReq.BaseReq.CurrentUser!;
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

        #region DownloadDestroyedBoxPDF Controller
        [HttpPost]
        [Route("DownloadDestroyedBoxPDF")]
        public DownloadDestroyedBoxPDFRes DownloadDestroyedBoxPDF(DownloadDestroyedBoxPDFReq downloadDestroyedBoxPDFReq)
        {
            DownloadDestroyedBoxPDFRes response = new()
            {
                Req = downloadDestroyedBoxPDFReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = downloadDestroyedBoxPDFReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "DownloadDestroyedBoxPDF",
                UserName = downloadDestroyedBoxPDFReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(downloadDestroyedBoxPDFReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : downloadDestroyedBoxPDFReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(downloadDestroyedBoxPDFReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : downloadDestroyedBoxPDFReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(downloadDestroyedBoxPDFReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(downloadDestroyedBoxPDFReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(DownloadPDFReq.BaseReq.CurrentEntity)} and {nameof(DownloadPDFReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(downloadDestroyedBoxPDFReq.BaseReq.CurrentEntity) ? String.Empty : downloadDestroyedBoxPDFReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(downloadDestroyedBoxPDFReq.BaseReq.CurrentBranch) ? String.Empty : downloadDestroyedBoxPDFReq.BaseReq.CurrentBranch;

                LogInfo("DownloadDestroyedBoxPDF Has been called with the following Request", correlationInfo);
                LogInfoJson(downloadDestroyedBoxPDFReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(downloadDestroyedBoxPDFReq) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateConfiguration call", correlationInfo);

                    response.Resp = oBLL.DownloadDestroyedBoxPDF(downloadDestroyedBoxPDFReq);

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

                    LogInfo("DownloadDestroyedBoxPDF Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the DownloadDestroyedBoxPDF is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : downloadDestroyedBoxPDFReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : downloadDestroyedBoxPDFReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : downloadDestroyedBoxPDFReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : downloadDestroyedBoxPDFReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = downloadDestroyedBoxPDFReq.BaseReq.CorrelationId!;
                response.WebResp.User = downloadDestroyedBoxPDFReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = downloadDestroyedBoxPDFReq.BaseReq.CorrelationId!;
                response.WebResp.User = downloadDestroyedBoxPDFReq.BaseReq.CurrentUser!;
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

        #region UpdateContainer Controller
        [HttpPost]
        [Route("UpdateContainer")]
        public UpdateContainerRes UpdateContainer(UpdateContainerReq updateContainerReq)
        {
            UpdateContainerRes response = new()
            {
                Req = updateContainerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateContainerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateContainer",
                UserName = updateContainerReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateContainerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateContainerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateContainerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateContainerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateContainerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateContainerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateContainerReq.BaseReq.CurrentEntity)} and {nameof(updateContainerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateContainerReq.BaseReq.CurrentEntity) ? String.Empty : updateContainerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateContainerReq.BaseReq.CurrentBranch) ? String.Empty : updateContainerReq.BaseReq.CurrentBranch;

                LogInfo("UpdateContainer Has been called with the following Request", correlationInfo);
                LogInfoJson(updateContainerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateContainerReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateContainerReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateContainer call", correlationInfo);

                    response.Resp = oBLL.UpdateContainer(updateContainerReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Container");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateContainer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateContainerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateContainerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateContainerReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateContainerStatus Controller
        [HttpPost]
        [Route("UpdateContainerStatus")]
        public UpdateContainerStatusRes UpdateContainerStatus(UpdateContainerStatusReq updateContainerStatusReq)
        {
            UpdateContainerStatusRes response = new()
            {
                Req = updateContainerStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateContainerStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateContainerStatus",
                UserName = updateContainerStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateContainerStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateContainerStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateContainerStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateContainerStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateContainerStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateContainerStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateContainerStatusReq.BaseReq.CurrentEntity)} and {nameof(updateContainerStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateContainerStatusReq.BaseReq.CurrentEntity) ? String.Empty : updateContainerStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateContainerStatusReq.BaseReq.CurrentBranch) ? String.Empty : updateContainerStatusReq.BaseReq.CurrentBranch;

                LogInfo("UpdateContainerStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(updateContainerStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateContainerStatusReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateContainerStatusReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateContainerStatus call", correlationInfo);

                    response.Resp = oBLL.UpdateContainerStatus(updateContainerStatusReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update ContainerStatus");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateContainerStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateContainerStatus is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateContainerStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateContainerStatusReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateContainerType Controller
        [HttpPost]
        [Route("UpdateContainerType")]
        public UpdateContainerTypeRes UpdateContainerType(UpdateContainerTypeReq updateContainerTypeReq)
        {
            UpdateContainerTypeRes response = new()
            {
                Req = updateContainerTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateContainerTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateContainerType",
                UserName = updateContainerTypeReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateContainerTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateContainerTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateContainerTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateContainerTypeReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateContainerTypeReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateContainerTypeReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateContainerTypeReq.BaseReq.CurrentEntity)} and {nameof(updateContainerTypeReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateContainerTypeReq.BaseReq.CurrentEntity) ? String.Empty : updateContainerTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateContainerTypeReq.BaseReq.CurrentBranch) ? String.Empty : updateContainerTypeReq.BaseReq.CurrentBranch;

                LogInfo("UpdateContainerType Has been called with the following Request", correlationInfo);
                LogInfoJson(updateContainerTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateContainerTypeReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateContainerTypeReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateContainerType call", correlationInfo);

                    response.Resp = oBLL.UpdateContainerType(updateContainerTypeReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update ContainerType");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateContainerType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateContainerType is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateContainerTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateContainerTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateContainerTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateContainerTypeReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateContainerTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateContainerTypeReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateContainerTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateContainerTypeReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateEntity Controller
        [HttpPost]
        [Route("UpdateEntity")]
        public UpdateEntityRes UpdateEntity(UpdateEntityReq updateEntityReq)
        {
            UpdateEntityRes response = new()
            {
                Req = updateEntityReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateEntityReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateEntity",
                UserName = updateEntityReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateEntityReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateEntityReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateEntityReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateEntityReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateEntityReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateEntityReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateEntityReq.BaseReq.CurrentEntity)} and {nameof(updateEntityReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateEntityReq.BaseReq.CurrentEntity) ? String.Empty : updateEntityReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateEntityReq.BaseReq.CurrentBranch) ? String.Empty : updateEntityReq.BaseReq.CurrentBranch;

                LogInfo("UpdateEntity Has been called with the following Request", correlationInfo);
                LogInfoJson(updateEntityReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateEntityReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateEntityReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateEntity call", correlationInfo);

                    response.Resp = oBLL.UpdateEntity(updateEntityReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Entity");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateEntity Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateEntity is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateEntityReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateEntityReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateEntityReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateEntityReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateEntityReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateCurrentContainerFileRelationship Controller
        [HttpPost]
        [Route("UpdateCurrentContainerFileRelationship")]
        public UpdateCurrentContainerFileRelationshipRes UpdateCurrentContainerFileRelationship(UpdateCurrentContainerFileRelationshipReq updateCurrentContainerFileRelationshipReq)
        {
            UpdateCurrentContainerFileRelationshipRes response = new()
            {
                Req = updateCurrentContainerFileRelationshipReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateCurrentContainerFileRelationshipReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateCurrentContainerFileRelationship",
                UserName = updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateCurrentContainerFileRelationshipReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateCurrentContainerFileRelationshipReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity)} and {nameof(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity) ? String.Empty : updateCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch) ? String.Empty : updateCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch;

                LogInfo("UpdateCurrentContainerFileRelationship Has been called with the following Request", correlationInfo);
                LogInfoJson(updateCurrentContainerFileRelationshipReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateCurrentContainerFileRelationshipReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateCurrentContainerFileRelationshipReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateCurrentContainerFileRelationship call", correlationInfo);

                    response.Resp = oBLL.UpdateCurrentContainerFileRelationship(updateCurrentContainerFileRelationshipReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update CurrentContainerFileRelationship");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCustomer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateCurrentContainerFileRelationshipReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateCurrentContainerFileRelationshipReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateCurrentContainerFileRelationshipReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateCurrentContainerFileRelationshipReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateCurrentContainerFileRelationshipReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateFile Controller
        [HttpPost]
        [Route("UpdateFile")]
        public UpdateFileRes UpdateFile(UpdateFileReq updateFileReq)
        {
            UpdateFileRes response = new()
            {
                Req = updateFileReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateFileReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateFile",
                UserName = updateFileReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateFileReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateFileReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateFileReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateFileReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateFileReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateFileReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateFileReq.BaseReq.CurrentEntity)} and {nameof(updateFileReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateFileReq.BaseReq.CurrentEntity) ? String.Empty : updateFileReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateFileReq.BaseReq.CurrentBranch) ? String.Empty : updateFileReq.BaseReq.CurrentBranch;

                LogInfo("UpdateFile Has been called with the following Request", correlationInfo);
                LogInfoJson(updateFileReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateFileReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateFileReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateFile call", correlationInfo);

                    response.Resp = oBLL.UpdateFile(updateFileReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update File");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateFile Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateFile is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateFileReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateFileReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateFileReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateFileName Controller
        [HttpPost]
        [Route("UpdateFileName")]
        public UpdateFileNameRes UpdateFileName(UpdateFileNameReq updateFileNameReq)
        {
            UpdateFileNameRes response = new()
            {
                Req = updateFileNameReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateFileNameReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateFileName",
                UserName = updateFileNameReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateFileNameReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateFileNameReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateFileNameReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateFileNameReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateFileNameReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateFileNameReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateFileNameReq.BaseReq.CurrentEntity)} and {nameof(updateFileNameReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateFileNameReq.BaseReq.CurrentEntity) ? String.Empty : updateFileNameReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateFileNameReq.BaseReq.CurrentBranch) ? String.Empty : updateFileNameReq.BaseReq.CurrentBranch;

                LogInfo("UpdateFileName Has been called with the following Request", correlationInfo);
                LogInfoJson(updateFileNameReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateFileNameReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateFileNameReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateFileName call", correlationInfo);

                    response.Resp = oBLL.UpdateFileName(updateFileNameReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update FileName");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateFileName Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateFileName is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateFileNameReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateFileNameReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateFileNameReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateFileNameReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateFileNameReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileNameReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateFileNameReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileNameReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateFileStatus Controller
        [HttpPost]
        [Route("UpdateFileStatus")]
        public UpdateFileStatusRes UpdateFileStatus(UpdateFileStatusReq updateFileStatusReq)
        {
            UpdateFileStatusRes response = new()
            {
                Req = updateFileStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateFileStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateFileStatus",
                UserName = updateFileStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateFileStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateFileStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateFileStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateFileStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateFileStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateFileStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateFileStatusReq.BaseReq.CurrentEntity)} and {nameof(updateFileStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateFileStatusReq.BaseReq.CurrentEntity) ? String.Empty : updateFileStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateFileStatusReq.BaseReq.CurrentBranch) ? String.Empty : updateFileStatusReq.BaseReq.CurrentBranch;

                LogInfo("UpdateFileStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(updateFileStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateFileStatusReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateFileStatusReq.Id }
                    };
                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateFileStatus call", correlationInfo);

                    response.Resp = oBLL.UpdateFileStatus(updateFileStatusReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update FileStatus");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    LogInfo("UpdateFileStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateFileStatus is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateFileStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateFileStatusReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateFileType Controller
        [HttpPost]
        [Route("UpdateFileType")]
        public UpdateFileTypeRes UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
        {
            UpdateFileTypeRes response = new()
            {
                Req = updateFileTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateFileTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL="UpdateFiletype",
                UserName = updateFileTypeReq.BaseReq.CurrentUser

            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateFileTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateFileTypeReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateFileTypeReq.BaseReq.CurrentEntity)} and {nameof(updateFileTypeReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : updateFileTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : updateFileTypeReq.BaseReq.CurrentBranch;

                LogInfo("UpdateFiletype Has been called with the following Request", correlationInfo);
                LogInfoJson(updateFileTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateFileTypeReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateFileTypeReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);
                    LogInfo("Start of UpdateFiletype call", correlationInfo);


                    response.Resp = oBLL.UpdateFileType(updateFileTypeReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update FileType");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateFiletype Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateFiletype is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateFileTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateFileTypeReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateStatus Controller
        [HttpPost]
        [Route("UpdateStatus")]
        public UpdateStatusRes UpdateStatus(UpdateStatusReq updateStatusReq)
        {
            UpdateStatusRes response = new()
            {
                Req = updateStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateStatus",
                UserName = updateStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateStatusReq.BaseReq.CurrentEntity)} and {nameof(updateStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateStatusReq.BaseReq.CurrentEntity) ? String.Empty : updateStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateStatusReq.BaseReq.CurrentBranch) ? String.Empty : updateStatusReq.BaseReq.CurrentBranch;

                LogInfo("UpdateStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(updateStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateStatusReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateStatusReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateStatus call", correlationInfo);

                    response.Resp = oBLL.UpdateStatus(updateStatusReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Status");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateStatus is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateStatusReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
        }
        #endregion

        #region UpdateUserInteraction Controller
        [HttpPost]
        [Route("UpdateUserInteraction")]
        public UpdateUserInteractionRes UpdateUserInteraction(UpdateUserInteractionReq updateUserInteractionReq)
        {
            UpdateUserInteractionRes response = new()
            {
                Req = updateUserInteractionReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateUserInteractionReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateUserInteraction",
                UserName = updateUserInteractionReq.BaseReq.CurrentUser
            };


            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateUserInteractionReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateUserInteractionReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateUserInteractionReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateUserInteractionReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateUserInteractionReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateUserInteractionReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateUserInteractionReq.BaseReq.CurrentEntity)} and {nameof(updateUserInteractionReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateUserInteractionReq.BaseReq.CurrentEntity) ? String.Empty : updateUserInteractionReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateUserInteractionReq.BaseReq.CurrentBranch) ? String.Empty : updateUserInteractionReq.BaseReq.CurrentBranch;

                LogInfo("UpdateUserInteraction Has been called with the following Request", correlationInfo);
                LogInfoJson(updateUserInteractionReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateUserInteractionReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateUserInteractionReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateUserInteraction call", correlationInfo);

                    response.Resp = oBLL.UpdateUserInteraction(updateUserInteractionReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update UserInteraction");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateUserInteraction Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetUpdateUserInteractionCustomer is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateUserInteractionReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateUserInteractionReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateUserInteractionReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateUserInteractionReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo("GetUpdateUserInteractionCustomer Has Replied with the Following response", correlationInfo);
                LogInfoJson(response, correlationInfo);
                LogInfo("Calling the GetUpdateUserInteractionCustomer is completed", correlationInfo);


                return response;
            }
            catch (SGBLInternalServerException ex)
            {
                response.WebResp.CorrelationId = updateUserInteractionReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateUserInteractionReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                //this was added in case correlation Id was invalid(null or Empty)
                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                //this was added in case Username was invalid(null or Empty)
                correlationInfo.UserName = response.WebResp.User;

                //don't forget to change status code in case of exception
                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateUserInteractionReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateUserInteractionReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateUsers Controller
        [HttpPost]
        [Route("UpdateUsers")]
        public UpdateUsersRes UpdateUsers(UpdateUsersReq updateUsersReq)
        {
            UpdateUsersRes response = new()
            {
                Req = updateUsersReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateUsersReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateUsers",
                UserName = updateUsersReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateUsersReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateUsersReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateUsersReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateUsersReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateUsersReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateUsersReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateUsersReq.BaseReq.CurrentEntity)} and {nameof(updateUsersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateUsersReq.BaseReq.CurrentEntity) ? String.Empty : updateUsersReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateUsersReq.BaseReq.CurrentBranch) ? String.Empty : updateUsersReq.BaseReq.CurrentBranch;

                LogInfo("UpdateUsers Has been called with the following Request", correlationInfo);
                LogInfoJson(updateUsersReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateUsersReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateUsersReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateUsers call", correlationInfo);

                    response.Resp = oBLL.UpdateUsers(updateUsersReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Users");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateUsers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GeUpdateUsers is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateUsersReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateUsersReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateUsersReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateUsersReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateUsersReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateUsersReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateUsersReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateUsersReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region ReceiveContainer Controller
        [HttpPost]
        [Route("ReceiveContainer")]
        public ReceiveContainerRes ReceiveContainer(ReceiveContainerReq receiveContainerReq)
        {
            ReceiveContainerRes response = new()
            {
                Req = receiveContainerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = receiveContainerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ReceiveContainer",
                UserName = receiveContainerReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(receiveContainerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : receiveContainerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(receiveContainerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : receiveContainerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(receiveContainerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(receiveContainerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(receiveContainerReq.BaseReq.CurrentEntity)} and {nameof(receiveContainerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(receiveContainerReq.BaseReq.CurrentEntity) ? String.Empty : receiveContainerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(receiveContainerReq.BaseReq.CurrentBranch) ? String.Empty : receiveContainerReq.BaseReq.CurrentBranch;

                LogInfo("ReceiveContainer Has been called with the following Request", correlationInfo);
                LogInfoJson(receiveContainerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(receiveContainerReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, receiveContainerReq.ContainerId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of ReceiveContainer call", correlationInfo);

                    response.Resp = oBLL.ReceiveContainer(receiveContainerReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Users");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("ReceiveContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the ReceiveContainer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : receiveContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : receiveContainerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : receiveContainerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : receiveContainerReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = receiveContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = receiveContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = receiveContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = receiveContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region UpdateWarehouse Controller
        [HttpPost]
        [Route("UpdateWarehouse")]
        public UpdateWarehouseRes UpdateWarehouse(UpdateWarehouseReq updateWarehouseReq)
        {
            UpdateWarehouseRes response = new()
            {
                Req = updateWarehouseReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateWarehouseReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateWarehouse",
                UserName = updateWarehouseReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateWarehouseReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateWarehouseReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateWarehouseReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateWarehouseReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateWarehouseReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateWarehouseReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateWarehouseReq.BaseReq.CurrentEntity)} and {nameof(updateWarehouseReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateWarehouseReq.BaseReq.CurrentEntity) ? String.Empty : updateWarehouseReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateWarehouseReq.BaseReq.CurrentBranch) ? String.Empty : updateWarehouseReq.BaseReq.CurrentBranch;

                LogInfo("UpdateWarehouse Has been called with the following Request", correlationInfo);
                LogInfoJson(updateWarehouseReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateWarehouseReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_CODE, updateWarehouseReq.Code }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateWarehouse call", correlationInfo);

                    response.Resp = oBLL.UpdateWarehouse(updateWarehouseReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Warehouse");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetCustomer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateWarehouseReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateWarehouseReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region ValidateCustomer Controller
        [HttpPost]
        [Route("ValidateCustomer")]
        public ValidateCustomerRes ValidateCustomer(ValidateCustomerReq validateCustomerReq)
        {
            ValidateCustomerRes response = new()
            {
                Req = validateCustomerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = validateCustomerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ValidateCustomer",
                UserName = validateCustomerReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(validateCustomerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : validateCustomerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(validateCustomerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : validateCustomerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(validateCustomerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(validateCustomerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(validateCustomerReq.BaseReq.CurrentEntity)} and {nameof(validateCustomerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(validateCustomerReq.BaseReq.CurrentEntity) ? String.Empty : validateCustomerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(validateCustomerReq.BaseReq.CurrentBranch) ? String.Empty : validateCustomerReq.BaseReq.CurrentBranch;

                LogInfo("ValidateCustomer Has been called with the following Request", correlationInfo);
                LogInfoJson(validateCustomerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(validateCustomerReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, validateCustomerReq.Id }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of ValidateCustomer call", correlationInfo);

                    response.Resp = oBLL.ValidateCustomer(validateCustomerReq);

                    if (String.IsNullOrEmpty(response.Resp))
                    {
                        throw new SGBLInternalServerException($"Failed to validate customer");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("ValidateCustomer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the ValidateCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : validateCustomerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : validateCustomerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : validateCustomerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : validateCustomerReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = String.Empty;

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
                response.WebResp.CorrelationId = validateCustomerReq.BaseReq.CorrelationId!;
                response.WebResp.User = validateCustomerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = String.Empty;

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = validateCustomerReq.BaseReq.CorrelationId!;
                response.WebResp.User = validateCustomerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = String.Empty;

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region RemoveFileFromContainer Controller
        [HttpPost]
        [Route("RemoveFileFromContainer")]
        public RemoveFileFromContainerRes RemoveFileFromContainer(RemoveFileFromContainerReq removeFileFromContainerReq)
        {
            RemoveFileFromContainerRes response = new()
            {
                Req = removeFileFromContainerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = removeFileFromContainerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "RemoveFileFromContainer",
                UserName = removeFileFromContainerReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(removeFileFromContainerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : removeFileFromContainerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(removeFileFromContainerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : removeFileFromContainerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(removeFileFromContainerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(removeFileFromContainerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(removeFileFromContainerReq.BaseReq.CurrentEntity)} and {nameof(removeFileFromContainerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(removeFileFromContainerReq.BaseReq.CurrentEntity) ? String.Empty : removeFileFromContainerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(removeFileFromContainerReq.BaseReq.CurrentBranch) ? String.Empty : removeFileFromContainerReq.BaseReq.CurrentBranch;

                LogInfo("RemoveFileFromContainer Has been called with the following Request", correlationInfo);
                LogInfoJson(removeFileFromContainerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(removeFileFromContainerReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_ID_LIST, new List<Int32> {removeFileFromContainerReq.FileId, removeFileFromContainerReq.ContainerId } },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of RemoveFileFromContainer call", correlationInfo);

                    response.Resp = oBLL.RemoveFileFromContainer(removeFileFromContainerReq);

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("RemoveFileFromContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the RemoveFileFromContainer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : removeFileFromContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : removeFileFromContainerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : removeFileFromContainerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : removeFileFromContainerReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = removeFileFromContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = removeFileFromContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = removeFileFromContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = removeFileFromContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region DeleteFile Controller
        [HttpPost]
        [Route("DeleteFile")]
        public DeleteFileRes DeleteFile(DeleteFileReq deleteFileReq)
        {
            DeleteFileRes response = new()
            {
                Req = deleteFileReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = deleteFileReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "DeleteFile",
                UserName = deleteFileReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(deleteFileReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : deleteFileReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(deleteFileReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : deleteFileReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(deleteFileReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(deleteFileReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(deleteFileReq.BaseReq.CurrentEntity)} and {nameof(deleteFileReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(deleteFileReq.BaseReq.CurrentEntity) ? String.Empty : deleteFileReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(deleteFileReq.BaseReq.CurrentBranch) ? String.Empty : deleteFileReq.BaseReq.CurrentBranch;

                LogInfo("DeleteFile Has been called with the following Request", correlationInfo);
                LogInfoJson(deleteFileReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(deleteFileReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, deleteFileReq.FileId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of DeleteFile call", correlationInfo);

                    response.Resp = oBLL.DeleteFile(deleteFileReq);

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("DeleteFile Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the DeleteFile is completed", correlationInfo);
                }
                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : deleteFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : deleteFileReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : deleteFileReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : deleteFileReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = deleteFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = deleteFileReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = deleteFileReq.BaseReq.CorrelationId!;
                response.WebResp.User = deleteFileReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region EditFileStatus Controller
        [HttpPost]
        [Route("EditFileStatus")]
        public EditFileStatusRes EditFileStatus(EditFileStatusReq editFileStatusReq)
        {
            EditFileStatusRes response = new()
            {
                Req = editFileStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = editFileStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "EditFileStatus",
                UserName = editFileStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(editFileStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : editFileStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(editFileStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : editFileStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(editFileStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(editFileStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(editFileStatusReq.BaseReq.CurrentEntity)} and {nameof(editFileStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(editFileStatusReq.BaseReq.CurrentEntity) ? String.Empty : editFileStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(editFileStatusReq.BaseReq.CurrentBranch) ? String.Empty : editFileStatusReq.BaseReq.CurrentBranch;

                LogInfo("EditFileStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(editFileStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(editFileStatusReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, editFileStatusReq.FileId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of EditFileStatus call", correlationInfo);

                    response.Resp = oBLL.EditFileStatus(editFileStatusReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException("Failed editing the file status of file Id: " + editFileStatusReq.FileId);
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("EditFileStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the EditFileStatus is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : editFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : editFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : editFileStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : editFileStatusReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = editFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = editFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = editFileStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = editFileStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region EditContainerStatus Controller
        [HttpPost]
        [Route("EditContainerStatus")]
        public EditContainerStatusRes EditContainerStatus(EditContainerStatusReq editContainerStatusReq)
        {
            EditContainerStatusRes response = new()
            {
                Req = editContainerStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = editContainerStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "EditContainerStatus",
                UserName = editContainerStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(editContainerStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : editContainerStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(editContainerStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : editContainerStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(editContainerStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(editContainerStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(editContainerStatusReq.BaseReq.CurrentEntity)} and {nameof(editContainerStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(editContainerStatusReq.BaseReq.CurrentEntity) ? String.Empty : editContainerStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(editContainerStatusReq.BaseReq.CurrentBranch) ? String.Empty : editContainerStatusReq.BaseReq.CurrentBranch;

                LogInfo("EditContainerStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(editContainerStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(editContainerStatusReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, editContainerStatusReq.ContainerId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of EditContainerStatus call", correlationInfo);

                    response.Resp = oBLL.EditContainerStatus(editContainerStatusReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException("Failed editing the container status of container Id: " + editContainerStatusReq.ContainerId);
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;
                    response.Req = editContainerStatusReq;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("EditContainerStatus Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the EditContainerStatus is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : editContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : editContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : editContainerStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : editContainerStatusReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = editContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = editContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = editContainerStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = editContainerStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetSentContainers Controller
        [HttpPost]
        [Route("GetSentContainers")]
        public GetSentContainersRes GetSentContainers(GetSentContainersReq getSentContainersReq)
        {
            GetSentContainersRes response = new()
            {
                Req = getSentContainersReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getSentContainersReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetSentContainers",
                UserName = getSentContainersReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getSentContainersReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getSentContainersReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getSentContainersReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getSentContainersReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getSentContainersReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getSentContainersReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getSentContainersReq.BaseReq.CurrentEntity)} and {nameof(getSentContainersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getSentContainersReq.BaseReq.CurrentEntity) ? String.Empty : getSentContainersReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getSentContainersReq.BaseReq.CurrentBranch) ? String.Empty : getSentContainersReq.BaseReq.CurrentBranch;

                LogInfo("GetSentContainers Has been called with the following Request", correlationInfo);
                LogInfoJson(getSentContainersReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getSentContainersReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetSentContainers(getSentContainersReq);

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

                    LogInfo("GetSentContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetSentContainers is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getSentContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getSentContainersReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getSentContainersReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getSentContainersReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getSentContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getSentContainersReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getSentContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = getSentContainersReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetWarehouseContainers Controller
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
        #endregion

        #region Get Entity Container By Status
        [HttpPost]
        [Route("GetEntityContainerByStatus")]
        public GetEntityContainerByStatusResp GetEntityContainerByStatus(GetEntityContainerByStatusReq getEntityContainerByStatusReq)
        {
            GetEntityContainerByStatusResp response = new()
            {
                Req = getEntityContainerByStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getEntityContainerByStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetEntityContainerByStatus",
                UserName = getEntityContainerByStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getEntityContainerByStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getEntityContainerByStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getEntityContainerByStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getEntityContainerByStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getEntityContainerByStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getEntityContainerByStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getEntityContainerByStatusReq.BaseReq.CurrentEntity)} and {nameof(getEntityContainerByStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getEntityContainerByStatusReq.BaseReq.CurrentEntity) ? String.Empty : getEntityContainerByStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getEntityContainerByStatusReq.BaseReq.CurrentBranch) ? String.Empty : getEntityContainerByStatusReq.BaseReq.CurrentBranch;

                LogInfo("GetEntityContainerByStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(getEntityContainerByStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getEntityContainerByStatusReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetEntityContainerByStatus(getEntityContainerByStatusReq);

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

                    LogInfo("GetSentContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetSentContainers is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getEntityContainerByStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getEntityContainerByStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getEntityContainerByStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getEntityContainerByStatusReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getEntityContainerByStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityContainerByStatusReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getEntityContainerByStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityContainerByStatusReq.BaseReq.CurrentUser!;
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
        #endregion

        #region Get RCA Container By Status
        [HttpPost]
        [Route("GetRCAContainerByStatus")]
        public GetRCAContainerByStatusResp GetRCAContainerByStatus(GetRCAContainerByStatusReq getRCAContainerByStatusReq)
        {
            GetRCAContainerByStatusResp response = new()
            {
                Req = getRCAContainerByStatusReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getRCAContainerByStatusReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetRCAContainerByStatus",
                UserName = getRCAContainerByStatusReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getRCAContainerByStatusReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getRCAContainerByStatusReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getRCAContainerByStatusReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getRCAContainerByStatusReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getRCAContainerByStatusReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getRCAContainerByStatusReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getRCAContainerByStatusReq.BaseReq.CurrentEntity)} and {nameof(getRCAContainerByStatusReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getRCAContainerByStatusReq.BaseReq.CurrentEntity) ? String.Empty : getRCAContainerByStatusReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getRCAContainerByStatusReq.BaseReq.CurrentBranch) ? String.Empty : getRCAContainerByStatusReq.BaseReq.CurrentBranch;

                LogInfo("GetRCAContainerByStatus Has been called with the following Request", correlationInfo);
                LogInfoJson(getRCAContainerByStatusReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getRCAContainerByStatusReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetRCAContainerByStatus(getRCAContainerByStatusReq);

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

                    LogInfo("GetSentContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetSentContainers is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getRCAContainerByStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getRCAContainerByStatusReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getRCAContainerByStatusReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getRCAContainerByStatusReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;
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
                response.WebResp.CorrelationId = getRCAContainerByStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getRCAContainerByStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = getRCAContainerByStatusReq.BaseReq.CorrelationId!;
                response.WebResp.User = getRCAContainerByStatusReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetContainerForEditByEntity Controller
        [HttpPost]
        [Route("GetContainerForEditByEntity")]
        public GetContainerForEditByEntityRes GetContainerForEditByEntity(GetContainerForEditByEntityReq getContainerForEditByEntityReq)
        {
            GetContainerForEditByEntityRes response = new()
            {
                Req = getContainerForEditByEntityReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerForEditByEntityReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerForEditByEntity",
                UserName = getContainerForEditByEntityReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerForEditByEntityReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerForEditByEntityReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerForEditByEntityReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerForEditByEntityReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerForEditByEntityReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerForEditByEntityReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerForEditByEntityReq.BaseReq.CurrentEntity)} and {nameof(getContainerForEditByEntityReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerForEditByEntityReq.BaseReq.CurrentEntity) ? String.Empty : getContainerForEditByEntityReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerForEditByEntityReq.BaseReq.CurrentBranch) ? String.Empty : getContainerForEditByEntityReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerForEditByEntity Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerForEditByEntityReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerForEditByEntityReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerForEditByEntity call", correlationInfo);

                    response.Resp = oBLL.GetContainerForEditByEntity(getContainerForEditByEntityReq);

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

                    LogInfo("GetContainerForEditByEntity Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerForEditByEntity is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerForEditByEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerForEditByEntityReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerForEditByEntityReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerForEditByEntityReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getContainerForEditByEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerForEditByEntityReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getContainerForEditByEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerForEditByEntityReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerForEditByRCA Controller
        [HttpPost]
        [Route("GetContainerForEditByRCA")]
        public GetContainerForEditByRCARes GetContainerForEditByRCA(GetContainerForEditByRCAReq getContainerForEditByRCAReq)
        {
            GetContainerForEditByRCARes response = new()
            {
                Req = getContainerForEditByRCAReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerForEditByRCAReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerForEditByRCA",
                UserName = getContainerForEditByRCAReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerForEditByRCAReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerForEditByRCAReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerForEditByRCAReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerForEditByRCAReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerForEditByRCAReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerForEditByRCAReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerForEditByRCAReq.BaseReq.CurrentEntity)} and {nameof(getContainerForEditByRCAReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerForEditByRCAReq.BaseReq.CurrentEntity) ? String.Empty : getContainerForEditByRCAReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerForEditByRCAReq.BaseReq.CurrentBranch) ? String.Empty : getContainerForEditByRCAReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerForEditByRCA Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerForEditByRCAReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerForEditByRCAReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerForEditByRCA call", correlationInfo);

                    response.Resp = oBLL.GetContainerForEditByRCA(getContainerForEditByRCAReq);

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

                    LogInfo("GetContainerForEditByRCA Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerForEditByRCA is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerForEditByRCAReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerForEditByRCAReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerForEditByRCAReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerForEditByRCAReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo("GetContainerForEditByRCA Has Replied with the Following response", correlationInfo);
                LogInfoJson(response, correlationInfo);
                LogInfo("Calling the GetContainerForEditByRCA is completed", correlationInfo);


                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = getContainerForEditByRCAReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerForEditByRCAReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

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
                response.WebResp.CorrelationId = getContainerForEditByRCAReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerForEditByRCAReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
        }
        #endregion

        #region AddFileToContainer Controller
        [HttpPost]
        [Route("AddFileToContainer")]
        public AddFileToContainerRes AddFileToContainer(AddFileToContainerReq addFileToContainerReq)
        {
            AddFileToContainerRes response = new()
            {
                Req = addFileToContainerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = addFileToContainerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "AddFileToContainer",
                UserName = addFileToContainerReq.BaseReq.CurrentUser
            };


            try
            {
                String CorrelationId = String.IsNullOrEmpty(addFileToContainerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : addFileToContainerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(addFileToContainerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : addFileToContainerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(addFileToContainerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(addFileToContainerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(addFileToContainerReq.BaseReq.CurrentEntity)} and {nameof(addFileToContainerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(addFileToContainerReq.BaseReq.CurrentEntity) ? String.Empty : addFileToContainerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(addFileToContainerReq.BaseReq.CurrentBranch) ? String.Empty : addFileToContainerReq.BaseReq.CurrentBranch;

                LogInfo("AddFileToContainer Has been called with the following Request", correlationInfo);
                LogInfoJson(addFileToContainerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(addFileToContainerReq) },
                        { DataIntegrityCheckFunctions.IS_NEGATIVE, addFileToContainerReq.ContainerId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of AddFileToContainer call", correlationInfo);

                    response.Resp = oBLL.AddFileToContainer(addFileToContainerReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed To Add File To Container: {addFileToContainerReq.ContainerId}");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("AddFileToContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetAddFileToContainerCustomer is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : addFileToContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : addFileToContainerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : addFileToContainerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : addFileToContainerReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = new();

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
                response.WebResp.CorrelationId = addFileToContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = addFileToContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = addFileToContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = addFileToContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetAllBranches Controller
        [HttpPost]
        [Route("GetAllBranches")]
        public GetAllBranchesRes GetAllBranches(GetAllBranchesReq getAllBranchesReq)
        {
            GetAllBranchesRes response = new()
            {
                Req = getAllBranchesReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getAllBranchesReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllBranches",
                UserName = getAllBranchesReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getAllBranchesReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getAllBranchesReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getAllBranchesReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getAllBranchesReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getAllBranchesReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getAllBranchesReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getAllBranchesReq.BaseReq.CurrentEntity)} and {nameof(getAllBranchesReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getAllBranchesReq.BaseReq.CurrentEntity) ? String.Empty : getAllBranchesReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getAllBranchesReq.BaseReq.CurrentBranch) ? String.Empty : getAllBranchesReq.BaseReq.CurrentBranch;

                LogInfo("GetAllBranches Has been called with the following Request", correlationInfo);
                LogInfoJson(getAllBranchesReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getAllBranchesReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetAllBranches call", correlationInfo);

                    response.Resp = oBLL.GetAllBranches(getAllBranchesReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException("No company have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetAllBranches Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetGetAllBranchesCustomer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getAllBranchesReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getAllBranchesReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getAllBranchesReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getAllBranchesReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getAllBranchesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getAllBranchesReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getAllBranchesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getAllBranchesReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetAllSequences Controller
        [HttpPost]
        [Route("GetAllSequences")]
        public GetAllSequencesRes GetAllSequences(GetAllSequencesReq getAllSequencesReq)
        {
            GetAllSequencesRes response = new()
            {
                Req = getAllSequencesReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getAllSequencesReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllSequences",
                UserName = getAllSequencesReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getAllSequencesReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getAllSequencesReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getAllSequencesReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getAllSequencesReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getAllSequencesReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getAllSequencesReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getAllSequencesReq.BaseReq.CurrentEntity)} and {nameof(getAllSequencesReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getAllSequencesReq.BaseReq.CurrentEntity) ? String.Empty : getAllSequencesReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getAllSequencesReq.BaseReq.CurrentBranch) ? String.Empty : getAllSequencesReq.BaseReq.CurrentBranch;

                LogInfo("GetAllSequences Has been called with the following Request", correlationInfo);
                LogInfoJson(getAllSequencesReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getAllSequencesReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetAllSequences call", correlationInfo);

                    response.Resp = oBLL.GetAllSequences(getAllSequencesReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException("No Sequence have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetAllSequences Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetAllSequences is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getAllSequencesReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getAllSequencesReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getAllSequencesReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getAllSequencesReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getAllSequencesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getAllSequencesReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getAllSequencesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getAllSequencesReq.BaseReq.CurrentUser!;
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
        #endregion

        #region UpdateSequence Controller
        [HttpPost]
        [Route("UpdateSequence")]
        public UpdateSequenceRes UpdateSequence(UpdateSequenceReq updateSequenceReq)
        {
            UpdateSequenceRes response = new()
            {
                Req = updateSequenceReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = updateSequenceReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateSequence",
                UserName = updateSequenceReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(updateSequenceReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateSequenceReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(updateSequenceReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateSequenceReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(updateSequenceReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(updateSequenceReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(updateSequenceReq.BaseReq.CurrentEntity)} and {nameof(updateSequenceReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(updateSequenceReq.BaseReq.CurrentEntity) ? String.Empty : updateSequenceReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(updateSequenceReq.BaseReq.CurrentBranch) ? String.Empty : updateSequenceReq.BaseReq.CurrentBranch;

                LogInfo("GetAllConfigurations Has been called with the following Request", correlationInfo);
                LogInfoJson(updateSequenceReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateSequenceReq) },
                        { DataIntegrityCheckFunctions.IS_INVALID_UPDATE_ID, updateSequenceReq.SequenceId }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of UpdateSequence call", correlationInfo);

                    response.Resp = oBLL.UpdateSequence(updateSequenceReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to create or update Sequence");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("UpdateSequence Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the UpdateSequence is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateSequenceReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateSequenceReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateSequenceReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateSequenceReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

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
                response.WebResp.CorrelationId = updateSequenceReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateSequenceReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = updateSequenceReq.BaseReq.CorrelationId!;
                response.WebResp.User = updateSequenceReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = new();

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetContainersToBeDestroyed Controller
        [HttpPost]
        [Route("ContainersToBeDestroyed")]
        public GetContainersToBeDestroyedRes ContainersToBeDestroyed(ContainersToBeDestroyedReq containersToBeDestroyedReq)
        {
            GetContainersToBeDestroyedRes response = new()
            {
                Req = containersToBeDestroyedReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = containersToBeDestroyedReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ContainersToBeDestroyed",
                UserName = containersToBeDestroyedReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(containersToBeDestroyedReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : containersToBeDestroyedReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(containersToBeDestroyedReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : containersToBeDestroyedReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(containersToBeDestroyedReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(containersToBeDestroyedReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(containersToBeDestroyedReq.BaseReq.CurrentEntity)} and {nameof(containersToBeDestroyedReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(containersToBeDestroyedReq.BaseReq.CurrentEntity) ? String.Empty : containersToBeDestroyedReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(containersToBeDestroyedReq.BaseReq.CurrentBranch) ? String.Empty : containersToBeDestroyedReq.BaseReq.CurrentBranch;

                LogInfo("ContainersToBeDestroyed Has been called with the following Request", correlationInfo);
                LogInfoJson(containersToBeDestroyedReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(containersToBeDestroyedReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetContainersToBeDestroyed(containersToBeDestroyedReq);

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

                    LogInfo("GetSentContainers Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetSentContainers is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : containersToBeDestroyedReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : containersToBeDestroyedReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : containersToBeDestroyedReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : containersToBeDestroyedReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = containersToBeDestroyedReq.BaseReq.CorrelationId!;
                response.WebResp.User = containersToBeDestroyedReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = containersToBeDestroyedReq.BaseReq.CorrelationId!;
                response.WebResp.User = containersToBeDestroyedReq.BaseReq.CurrentUser!;
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
        #endregion

        #region DestroyContainers Controller
        [HttpPost]
        [Route("DestroyContainers")]
        public DestroyContainersRes DestroyContainers(DestroyContainersReq destroyContainersReq)
        {
            DestroyContainersRes response = new()
            {
                Req = destroyContainersReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = destroyContainersReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "DestroyContainers",
                UserName = destroyContainersReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(destroyContainersReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : destroyContainersReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(destroyContainersReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : destroyContainersReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(destroyContainersReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(destroyContainersReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(destroyContainersReq.BaseReq.CurrentEntity)} and {nameof(destroyContainersReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(destroyContainersReq.BaseReq.CurrentEntity) ? String.Empty : destroyContainersReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(destroyContainersReq.BaseReq.CurrentBranch) ? String.Empty : destroyContainersReq.BaseReq.CurrentBranch;

                LogInfo("DestroyContainers Has been called with the following Request", correlationInfo);
                LogInfoJson(destroyContainersReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(destroyContainersReq) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of DestroyContainerss call", correlationInfo);

                    response.Resp = oBLL.DestroyContainers(destroyContainersReq);

                    if (response.Resp == null)
                    {
                        throw new SGBLInternalServerException($"Failed to Destroy Containers");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("DestroyContainerss Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the DestroyContainerss is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : destroyContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : destroyContainersReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : destroyContainersReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : destroyContainersReq.BaseReq.CurrentBranch!;
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
            catch (SGBLInternalServerException ex)
            {
                response.WebResp.CorrelationId = destroyContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = destroyContainersReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = destroyContainersReq.BaseReq.CorrelationId!;
                response.WebResp.User = destroyContainersReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetActiveCompaniesOfUser Controller
        [HttpPost]
        [Route("GetActiveCompaniesOfUser")]
        public GetActiveCompaniesOfUserRes GetActiveCompaniesOfUser(GetActiveCompaniesOfUserReq getActiveCompaniesOfUserReq)
        {
            GetActiveCompaniesOfUserRes response = new()
            {
                Req = getActiveCompaniesOfUserReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getActiveCompaniesOfUserReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetActiveCompaniesOfUser",
                UserName = getActiveCompaniesOfUserReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getActiveCompaniesOfUserReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getActiveCompaniesOfUserReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getActiveCompaniesOfUserReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getActiveCompaniesOfUserReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getActiveCompaniesOfUserReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getActiveCompaniesOfUserReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getActiveCompaniesOfUserReq.BaseReq.CurrentEntity)} and {nameof(getActiveCompaniesOfUserReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getActiveCompaniesOfUserReq.BaseReq.CurrentEntity) ? String.Empty : getActiveCompaniesOfUserReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getActiveCompaniesOfUserReq.BaseReq.CurrentBranch) ? String.Empty : getActiveCompaniesOfUserReq.BaseReq.CurrentBranch;

                LogInfo("GetGetActiveCompaniesOfUser Has been called with the following Request", correlationInfo);
                LogInfoJson(getActiveCompaniesOfUserReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getActiveCompaniesOfUserReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetActiveCompaniesOfUser(getActiveCompaniesOfUserReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Active Entities have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetGetActiveCompaniesOfUser Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetGetActiveCompaniesOfUser is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getActiveCompaniesOfUserReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getActiveCompaniesOfUserReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getActiveCompaniesOfUserReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getActiveCompaniesOfUserReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getActiveCompaniesOfUserReq.BaseReq.CorrelationId!;
                response.WebResp.User = getActiveCompaniesOfUserReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NotFound;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = getActiveCompaniesOfUserReq.BaseReq.CorrelationId!;
                response.WebResp.User = getActiveCompaniesOfUserReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetEntityFileTypes Controller
        [HttpPost]
        [Route("GetEntityFileTypes")]
        public GetEntityFileTypesRes GetEntityFileTypes(GetEntityFileTypesReq getEntityFileTypesReq)
        {
            GetEntityFileTypesRes response = new()
            {
                Req = getEntityFileTypesReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getEntityFileTypesReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetEntityFileTypes",
                UserName = getEntityFileTypesReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getEntityFileTypesReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getEntityFileTypesReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getEntityFileTypesReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getEntityFileTypesReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getEntityFileTypesReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getEntityFileTypesReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getEntityFileTypesReq.BaseReq.CurrentEntity)} and {nameof(getEntityFileTypesReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getEntityFileTypesReq.BaseReq.CurrentEntity) ? String.Empty : getEntityFileTypesReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getEntityFileTypesReq.BaseReq.CurrentBranch) ? String.Empty : getEntityFileTypesReq.BaseReq.CurrentBranch;

                LogInfo("GetGetEntityFileTypes Has been called with the following Request", correlationInfo);
                LogInfoJson(getEntityFileTypesReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getEntityFileTypesReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetEntityFileTypes(getEntityFileTypesReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No Active Entities have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetGetEntityFileTypes Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetGetEntityFileTypes is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getEntityFileTypesReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getEntityFileTypesReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getEntityFileTypesReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getEntityFileTypesReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getEntityFileTypesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityFileTypesReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NotFound;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = getEntityFileTypesReq.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityFileTypesReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetIsEntityValidationPermittedForContainer Controller
        [HttpPost]
        [Route("GetIsEntityValidationPermittedForContainer")]
        public GetIsEntityValidationPermittedForContainerRes GetIsEntityValidationPermittedForContainer(GetIsEntityValidationPermittedForContainerReq getIsEntityValidationPermittedForContainerReq)
        {
            GetIsEntityValidationPermittedForContainerRes response = new()
            {
                Req = getIsEntityValidationPermittedForContainerReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getIsEntityValidationPermittedForContainerReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetIsEntityValidationPermittedForContainer",
                UserName = getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getIsEntityValidationPermittedForContainerReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getIsEntityValidationPermittedForContainerReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentEntity)} and {nameof(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentEntity) ? String.Empty : getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentBranch) ? String.Empty : getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentBranch;

                LogInfo("GetGetIsEntityValidationPermittedForContainer Has been called with the following Request", correlationInfo);
                LogInfoJson(getIsEntityValidationPermittedForContainerReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getIsEntityValidationPermittedForContainerReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetSentContainers call", correlationInfo);

                    response.Resp = oBLL.GetIsEntityValidationPermittedForContainer(getIsEntityValidationPermittedForContainerReq);

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetGetIsEntityValidationPermittedForContainer Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetGetIsEntityValidationPermittedForContainer is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getIsEntityValidationPermittedForContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = false;

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
                response.WebResp.CorrelationId = getIsEntityValidationPermittedForContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NotFound;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = false;

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = getIsEntityValidationPermittedForContainerReq.BaseReq.CorrelationId!;
                response.WebResp.User = getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = false;

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetEntityFilesByFileType Controller
        [HttpPost]
        [Route("GetEntityFilesByFileType")]
        public GetEntityFilesByFileTypeRes GetEntityFilesByFileType(GetEntityFilesByFileTypeReq getEntityFilesByFileType)
        {
            GetEntityFilesByFileTypeRes response = new()
            {
                Req = getEntityFilesByFileType
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getEntityFilesByFileType.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetEntityFilesByFileType",
                UserName = getEntityFilesByFileType.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getEntityFilesByFileType.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getEntityFilesByFileType.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getEntityFilesByFileType.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getEntityFilesByFileType.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getEntityFilesByFileType.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getEntityFilesByFileType.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getEntityFilesByFileType.BaseReq.CurrentEntity)} and {nameof(getEntityFilesByFileType.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getEntityFilesByFileType.BaseReq.CurrentEntity) ? String.Empty : getEntityFilesByFileType.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getEntityFilesByFileType.BaseReq.CurrentBranch) ? String.Empty : getEntityFilesByFileType.BaseReq.CurrentBranch;

                LogInfo("GetEntityFilesByFileType Has been called with the following Request", correlationInfo);
                LogInfoJson(getEntityFilesByFileType, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getEntityFilesByFileType) }
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetEntityFilesByFileType call", correlationInfo);

                    response.Resp = oBLL.GetEntityFilesByFileType(getEntityFilesByFileType);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File have been found in our sytems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetEntityFilesByFileType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetEntityFilesByFileType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getEntityFilesByFileType.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getEntityFilesByFileType.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getEntityFilesByFileType.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getEntityFilesByFileType.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = getEntityFilesByFileType.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityFilesByFileType.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = getEntityFilesByFileType.BaseReq.CorrelationId!;
                response.WebResp.User = getEntityFilesByFileType.BaseReq.CurrentUser!;
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
        #endregion

        #region GetBranchFileType Controller
        [HttpPost]
        [Route("GetBranchFileType")]
        public GetBranchFileTypeRes GetBranchFileType(GetBranchFileTypeReq GetBranchFileTypeReq)
        {
            GetBranchFileTypeRes response = new()
            {
                Req = GetBranchFileTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = GetBranchFileTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetBranchFileType",
                UserName = GetBranchFileTypeReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(GetBranchFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : GetBranchFileTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(GetBranchFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : GetBranchFileTypeReq.BaseReq.CurrentUser;



                String CurrentEntity = String.IsNullOrEmpty(GetBranchFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : GetBranchFileTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(GetBranchFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : GetBranchFileTypeReq.BaseReq.CurrentBranch;

                LogInfo("GetBranchFileType Has been called with the following Request", correlationInfo);
                LogInfoJson(GetBranchFileTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
            {
                { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(GetBranchFileTypeReq) },
            };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetBranchFileType call", correlationInfo);

                    response.Resp = oBLL.GetBranchFileType(GetBranchFileTypeReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Types have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetBranchFileType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetBranchFileType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : GetBranchFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : GetBranchFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : GetBranchFileTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : GetBranchFileTypeReq.BaseReq.CurrentBranch!;
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
                response.WebResp.CorrelationId = GetBranchFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = GetBranchFileTypeReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = GetBranchFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = GetBranchFileTypeReq.BaseReq.CurrentUser!;
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
        #endregion

        #region GetContainerToNotifyWarehouse Controller RCA
        [HttpPost]
        [Route("GetContainerToNotifyWarehouse")]
        public GetContainerToNotifyWarehouseRes GetContainerToNotifyWarehouse(GetContainerToNotifyWarehouseReq getContainerToNotifyWarehouseReq)
        {
            GetContainerToNotifyWarehouseRes response = new()
            {
                Req = getContainerToNotifyWarehouseReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerToNotifyWarehouseReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerToNotifyWarehouse",
                UserName = getContainerToNotifyWarehouseReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerToNotifyWarehouseReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerToNotifyWarehouseReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerToNotifyWarehouseReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerToNotifyWarehouseReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerToNotifyWarehouseReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(getContainerToNotifyWarehouseReq.BaseReq.CurrentEntity)} and {nameof(getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(getContainerToNotifyWarehouseReq.BaseReq.CurrentEntity) ? String.Empty : getContainerToNotifyWarehouseReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch) ? String.Empty : getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerToNotifyWarehouse Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerToNotifyWarehouseReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerToNotifyWarehouseReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerToNotifyWarehouse call", correlationInfo);

                    response.Resp = oBLL.GetContainerToNotifyWarehouse(getContainerToNotifyWarehouseReq);

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

                    LogInfo("GetContainerToNotifyWarehouse Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerToNotifyWarehouse is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerToNotifyWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerToNotifyWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerToNotifyWarehouseReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo("GetContainerToNotifyWarehouse Has Replied with the Following response", correlationInfo);
                LogInfoJson(response, correlationInfo);
                LogInfo("Calling the GetContainerToNotifyWarehouse is completed", correlationInfo);


                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = getContainerToNotifyWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerToNotifyWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

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
                response.WebResp.CorrelationId = getContainerToNotifyWarehouseReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerToNotifyWarehouseReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);


                return response;
            }
        }
        #endregion

        #region GetContainerToNotifyWarehouseByEntity Controller Entity
        [HttpPost]
        [Route("GetContainerToNotifyWarehouseByEntity")]
        public GetContainerToNotifyWarehouseByEntityRes GetContainerToNotifyWarehouseByEntity(GetContainerToNotifyWarehouseByEntityReq getContainerToNotifyWarehouseByEntityReq)
        {
            GetContainerToNotifyWarehouseByEntityRes response = new()
            {
                Req = getContainerToNotifyWarehouseByEntityReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = getContainerToNotifyWarehouseByEntityReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetContainerToNotifyWarehouseByEntity",
                UserName = getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(getContainerToNotifyWarehouseByEntityReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : getContainerToNotifyWarehouseByEntityReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(GetContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentEntity)} and {nameof(GetContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = 
                    String.IsNullOrEmpty(getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentEntity) ? String.Empty : getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentEntity;
                String CurrentBranch = 
                    String.IsNullOrEmpty(getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch) ? String.Empty : getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch;

                LogInfo("GetContainerToNotifyWarehouseByEntity Has been called with the following Request", correlationInfo);
                LogInfoJson(getContainerToNotifyWarehouseByEntityReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(getContainerToNotifyWarehouseByEntityReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetContainerToNotifyWarehouseByEntity call", correlationInfo);

                    response.Resp = oBLL.GetContainerToNotifyWarehouseByEntity(getContainerToNotifyWarehouseByEntityReq);

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

                    LogInfo("GetContainerToNotifyWarehouseByEntity Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetContainerToNotifyWarehouseByEntity is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : getContainerToNotifyWarehouseByEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo("GetContainerToNotifyWarehouseByEntity Has Replied with the Following response", correlationInfo);
                LogInfoJson(response, correlationInfo);
                LogInfo("Calling the GetContainerToNotifyWarehouseByEntity is completed", correlationInfo);

                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = getContainerToNotifyWarehouseByEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

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
                response.WebResp.CorrelationId = getContainerToNotifyWarehouseByEntityReq.BaseReq.CorrelationId!;
                response.WebResp.User = getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.Message;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region NotifyContainersByIds
        [HttpPost]
        [Route("NotifyContainersByIds")]
        public NotifyContainersByIdsRes NotifyContainersByIds(NotifyContainersByIdsReq NotifyContainersByIdsReq)
        {
            NotifyContainersByIdsRes response = new NotifyContainersByIdsRes()
            {
                Req = NotifyContainersByIdsReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = NotifyContainersByIdsReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "NotifyContainersByIds",
                UserName = NotifyContainersByIdsReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(NotifyContainersByIdsReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : NotifyContainersByIdsReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(NotifyContainersByIdsReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : NotifyContainersByIdsReq.BaseReq.CurrentUser;

                if (String.IsNullOrEmpty(NotifyContainersByIdsReq.BaseReq.CurrentEntity) && String.IsNullOrEmpty(NotifyContainersByIdsReq.BaseReq.CurrentBranch))
                {
                    throw new SGBLBadRequestException($"{nameof(NotifyContainersByIdsReq.BaseReq.CurrentEntity)} and {nameof(GetContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch)} Cannot Be null or empty");
                }

                String CurrentEntity = String.IsNullOrEmpty(NotifyContainersByIdsReq.BaseReq.CurrentEntity) ? String.Empty : NotifyContainersByIdsReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(NotifyContainersByIdsReq.BaseReq.CurrentBranch) ? String.Empty : NotifyContainersByIdsReq.BaseReq.CurrentBranch;

                LogInfo("NotifyContainersByIds Has been called with the following Request", correlationInfo);
                LogInfoJson(NotifyContainersByIdsReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;


                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
                    {
                        { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(NotifyContainersByIdsReq) },
                    };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of NotifyContainersByIds call", correlationInfo);

                    oBLL.NotifyContainersByIds(NotifyContainersByIdsReq);

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("NotifyContainersByIds Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the NotifyContainersByIds is completed", correlationInfo);
                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : NotifyContainersByIdsReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : NotifyContainersByIdsReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : NotifyContainersByIdsReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : NotifyContainersByIdsReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;


                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo("NotifyContainersByIds Has Replied with the Following response", correlationInfo);
                LogInfoJson(response, correlationInfo);
                LogInfo("Calling the NotifyContainersByIds is completed", correlationInfo);


                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = NotifyContainersByIdsReq.BaseReq.CorrelationId!;
                response.WebResp.User = NotifyContainersByIdsReq.BaseReq.CurrentUser!;
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
                response.WebResp.CorrelationId = NotifyContainersByIdsReq.BaseReq.CorrelationId!;
                response.WebResp.User = NotifyContainersByIdsReq.BaseReq.CurrentUser!;
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

        #region Backfill missing PDF
        [HttpPost]
        [Route("BackfillMissingPDFs")]
        public BackfillMissingPDFsRes BackfillMissingPDFs(BackfillMissingPDFsReq backfillReq)
        {
            BackfillMissingPDFsRes response = new()
            {
                Req = backfillReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = backfillReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "BackfillMissingPDFs",
                UserName = backfillReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(backfillReq.BaseReq.CorrelationId)
                    ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty")
                    : backfillReq.BaseReq.CorrelationId;

                String CurrentUser = String.IsNullOrEmpty(backfillReq.BaseReq.CurrentUser)
                    ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty")
                    : backfillReq.BaseReq.CurrentUser;

                LogInfo("BackfillMissingPDFs Has been called with the following Request", correlationInfo);
                LogInfoJson(backfillReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Start of BackfillMissingPDFs call", correlationInfo);

                    response.Resp = oBLL.BackfillMissingPDFsForLegacyContainers(
                        CurrentUser,
                        backfillReq.FromDate,
                        backfillReq.ToDate
                    );

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("BackfillMissingPDFs Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the BackfillMissingPDFs is completed", correlationInfo);
                }

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId")
                    ? Guid.NewGuid().ToString()
                    : backfillReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser")
                    ? "BadUser"
                    : backfillReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;

                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                correlationInfo.UserName = response.WebResp.User;
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = backfillReq.BaseReq.CorrelationId!;
                response.WebResp.User = backfillReq.BaseReq.CurrentUser!;
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
    }
}

using System.Configuration;
using System.Data;
using System.Text;

using ALTERNA.ARCHIVING.BLL.ViewModels;
using ALTERNA.ARCHIVING.DAL;

using Azure.Core;

using Dapper;

using EmailSenderTokenGenerator;

using Newtonsoft.Json;

using static ALTERNA.ARCHIVING.DAL.DAL;
using static NLog.NLogUtil;

namespace ALTERNA.ARCHIVING.BLL
{
    public partial class BLL : IDisposable
    {
        #region ConvertIntListToString

        public String ConvertIntListToString(List<Int32> listInt) => String.Join(",", listInt);

        #endregion

        #region Disposable

        void IDisposable.Dispose()
        {
        }

        #endregion

        #region GetAllConfigurations

        public List<ArchiveConfiguration> GetAllConfigurations(GetAllConfigurationsReq getallConfigurationReq)
        {
            DAL.DAL iDAL = new();
            List<ArchiveConfiguration> Retlist = [];
            OnPreEventGetAllConfigurations?.Invoke(ref getallConfigurationReq);

            Retlist = iDAL.ExecuteQuery<ArchiveConfiguration>("usp_GetAllConfigurations", null,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetAllConfigurations?.Invoke(ref Retlist, ref getallConfigurationReq);
            return Retlist;
        }

        #endregion

        #region GetActiveConfigurations

        public List<ArchiveConfiguration> GetActiveConfigurations(GetActiveConfigurationsReq getactiveConfigurationReq)
        {
            DAL.DAL iDAL = new();
            List<ArchiveConfiguration> Retlist = [];
            OnPreEventGetActiveConfigurations?.Invoke(ref getactiveConfigurationReq);

            Retlist = iDAL.ExecuteQuery<ArchiveConfiguration>("usp_GetActiveConfigurations", null,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetActiveConfigurations?.Invoke(ref Retlist, ref getactiveConfigurationReq);
            return Retlist;
        }

        #endregion

        #region GetCompany

        public List<Company> GetCompany(GetCompanyReq getCompanyReq)
        {
            DAL.DAL iDAL = new();
            List<Company> Retlist = [];
            OnPreEventGetCompany?.Invoke(ref getCompanyReq);

            DynamicParameters param = new();
            param.Add("Codes", getCompanyReq.Codes);

            Retlist = iDAL.ExecuteQuery<Company>("usp_GetCompany", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetCompany?.Invoke(ref Retlist, ref getCompanyReq);
            return Retlist;
        }

        #endregion

        #region GetAllCompanies

        public List<Company> GetAllCompanies(GetAllCompaniesReq getAllCompaniesReq)
        {
            DAL.DAL iDAL = new();
            List<Company> Retlist = [];
            OnPreEventGetAllCompanies?.Invoke(ref getAllCompaniesReq);

            Retlist = iDAL.ExecuteQuery<Company>("usp_GetAllCompanies", null, CommandType.StoredProcedure,
                CommandDirection.Select);

            OnPostEventGetAllCompanies?.Invoke(ref Retlist, ref getAllCompaniesReq);

            return Retlist;
        }

        #endregion

        #region GetConfiguration

        public List<ArchiveConfiguration> GetConfiguration(GetConfigurationReq getConfigurationReq)
        {
            DAL.DAL iDAL = new();
            List<ArchiveConfiguration> Retlist = [];
            OnPreEventGetConfiguration?.Invoke(ref getConfigurationReq);

            DynamicParameters param = new();
            param.Add("SettingName", getConfigurationReq.SettingName);

            Retlist = iDAL.ExecuteQuery<ArchiveConfiguration>("usp_GetConfiguration", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetConfiguration?.Invoke(ref Retlist, ref getConfigurationReq);
            return Retlist;
        }

        #endregion

        #region GetContainer

        public List<Container> GetContainer(GetContainerReq getContainerReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainer?.Invoke(ref getContainerReq);

            DynamicParameters param = new();
            param.Add("Ids", ConvertIntListToString(getContainerReq.Ids));

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainer", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainer?.Invoke(ref Retlist, ref getContainerReq);
            return Retlist;
        }

        #endregion

        #region GetContainerByCode

        public List<Container> GetContainerByCode(GetContainerByCodeReq getContainerByCodeReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerByCode?.Invoke(ref getContainerByCodeReq);

            DynamicParameters param = new();
            param.Add("Codes", String.Join(",", getContainerByCodeReq.Codes));

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainerByCode", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerByCode?.Invoke(ref Retlist, ref getContainerByCodeReq);
            return Retlist;
        }

        #endregion

        #region GetContainerByStatus

        public List<Container> GetContainerByStatus(GetContainerByStatusReq getContainerByStatusReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerByStatus?.Invoke(ref getContainerByStatusReq);

            DynamicParameters param = new();
            param.Add("Status", getContainerByStatusReq.Status);
            param.Add("CompanyCodes", getContainerByStatusReq.BaseReq.CurrentBranch);
            param.Add("Entities", getContainerByStatusReq.BaseReq.CurrentEntity);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainerByStatus", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerByStatus?.Invoke(ref Retlist, ref getContainerByStatusReq);
            return Retlist;
        }

        #endregion

        #region GetBranchContainer

        public List<Container> GetBranchContainers(GetBranchContainerReq getBranchContainerReq)
        {
            DAL.DAL iDAL = new();
            List<Container> RetList = [];

            OnPreEventGetBranchContainer?.Invoke(ref getBranchContainerReq);

            DynamicParameters param = new();

            param.Add("FromDate", getBranchContainerReq.FromDate);
            param.Add("ToDate", getBranchContainerReq.ToDate);
            param.Add("StatusCode", getBranchContainerReq.StatusCode);
            param.Add("CompanyCodes", getBranchContainerReq.BaseReq.CurrentBranch);

            RetList = iDAL.ExecuteQuery<Container>("usp_GetBranchContainer", param, CommandType.StoredProcedure,
                CommandDirection.Select);


            OnPostEventGetBranchContainer?.Invoke(ref RetList, ref getBranchContainerReq);

            return RetList;
        }

        #endregion

        #region GetEntityContainers

        public List<Container> GetEntityContainers(GetEntityContainerReq getEntityContainerReq)
        {
            DAL.DAL iDAL = new();
            List<Container> RetList = [];

            OnPreEventGetEntityContainer?.Invoke(ref getEntityContainerReq);

            DynamicParameters param = new();

            param.Add("FromDate", getEntityContainerReq.FromDate);
            param.Add("ToDate", getEntityContainerReq.ToDate);
            param.Add("StatusCode", getEntityContainerReq.StatusCode);
            param.Add("CompanyCodes", getEntityContainerReq.BaseReq.CurrentBranch);

            RetList = iDAL.ExecuteQuery<Container>("usp_GetBranchContainer", param, CommandType.StoredProcedure,
                CommandDirection.Select);


            OnPostEventGetEntityContainer?.Invoke(ref RetList, ref getEntityContainerReq);

            return RetList;
        }

        #endregion

        #region GetContainerByEntityOrBranch

        public List<Container> GetContainerByEntityOrBranch(
            GetContainerByEntityOrBranchReq getContainerByEntityOrBranchReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerByEntityOrBranch?.Invoke(ref getContainerByEntityOrBranchReq);

            DynamicParameters param = new();
            param.Add("Entities", getContainerByEntityOrBranchReq.BaseReq.CurrentEntity);
            param.Add("CompanyCodes", getContainerByEntityOrBranchReq.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainerByEntityOrBranch", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetContainerByEntityOrBranch?.Invoke(ref Retlist, ref getContainerByEntityOrBranchReq);
            return Retlist;
        }

        #endregion

        #region GetCustomerFilesByCustomerId

        public List<ArchivedFile> GetCustomerFilesByCustomerId(
            GetCustomerFilesByCustomerIdReq getCustomerFilesByCustomerId)
        {
            DAL.DAL iDAL = new();
            List<ArchivedFile> Retlist = [];
            OnPreEventGetCustomerFilesByCustomerId?.Invoke(ref getCustomerFilesByCustomerId);

            DynamicParameters param = new();
            param.Add("CustomerId", getCustomerFilesByCustomerId.CustomerId);
            param.Add("Entities", getCustomerFilesByCustomerId.BaseReq.CurrentEntity);
            param.Add("CompanyCodes", getCustomerFilesByCustomerId.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<ArchivedFile>("usp_GetCustomerFilesByCustomerId", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetCustomerFilesByCustomerId?.Invoke(ref Retlist, ref getCustomerFilesByCustomerId);
            return Retlist;
        }

        #endregion

        #region GetContainerFiles

        public Container GetContainerFiles(GetContainerFilesReq getContainerFiles)
        {
            DAL.DAL iDAL = new();
            Container Ret = new();
            OnPreEventGetContainerFiles?.Invoke(ref getContainerFiles);

            DynamicParameters param = new();
            param.Add("ContainerId", getContainerFiles.ContainerId);
            param.Add("Entities", getContainerFiles.BaseReq.CurrentEntity);
            param.Add("CompanyCodes", getContainerFiles.BaseReq.CurrentBranch);

            Ret.Files = iDAL.ExecuteQuery<ArchivedFile>("usp_GetContainerFiles", param, CommandType.StoredProcedure,
                CommandDirection.Select);

            OnPostEventGetContainerFiles?.Invoke(ref Ret, ref getContainerFiles);
            return Ret;
        }

        #endregion

        #region GetGeneralFilesByFileType

        public List<ArchivedFile> GetGeneralFilesByFileType(GetGeneralFilesByFileTypeReq getGeneralFilesByFileType)
        {
            DAL.DAL iDAL = new();
            List<ArchivedFile> Retlist = [];
            OnPreEventGetGeneralFilesByFileType?.Invoke(ref getGeneralFilesByFileType);

            DynamicParameters param = new();
            param.Add("FileTypeCode", getGeneralFilesByFileType.FileTypeCode);
            param.Add("Entities", getGeneralFilesByFileType.BaseReq.CurrentEntity);
            param.Add("CompanyCodes", getGeneralFilesByFileType.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<ArchivedFile>("usp_GetGeneralFilesByFileType", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetGeneralFilesByFileType?.Invoke(ref Retlist, ref getGeneralFilesByFileType);
            return Retlist;
        }

        #endregion

        #region GetContainerByFileId

        public List<Container> GetContainerByFileId(GetContainerByFileIdReq getContainerByFileIdReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerByFileId?.Invoke(ref getContainerByFileIdReq);

            DynamicParameters param = new();
            param.Add("FileIds", ConvertIntListToString(getContainerByFileIdReq.FileIds));

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainerByFileId", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerByFileId?.Invoke(ref Retlist, ref getContainerByFileIdReq);
            return Retlist;
        }

        #endregion

        #region GetContainerStatus

        public List<ContainerStatus> GetContainerStatus(GetContainerStatusReq getContainerStatusReq)
        {
            DAL.DAL iDAL = new();
            List<ContainerStatus> Retlist = [];
            OnPreEventGetContainerStatus?.Invoke(ref getContainerStatusReq);

            DynamicParameters param = new();
            param.Add("Ids", ConvertIntListToString(getContainerStatusReq.Ids));

            Retlist = iDAL.ExecuteQuery<ContainerStatus>("usp_GetContainerStatus", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerStatus?.Invoke(ref Retlist, ref getContainerStatusReq);
            return Retlist;
        }

        #endregion

        #region GetContainerType

        public List<ContainerType> GetContainerType(GetContainerTypeReq getContainerTypeReq)
        {
            DAL.DAL iDAL = new();
            List<ContainerType> Retlist = [];
            OnPreEventGetContainerType?.Invoke(ref getContainerTypeReq);

            Retlist = iDAL.ExecuteQuery<ContainerType>("usp_GetContainerType", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerType?.Invoke(ref Retlist, ref getContainerTypeReq);
            return Retlist;
        }

        #endregion

        #region GetEntity

        public List<Entity> GetEntity(GetEntityReq getEntityReq)
        {
            DAL.DAL iDAL = new();
            List<Entity> Retlist = [];
            OnPreEventGetEntity?.Invoke(ref getEntityReq);

            Retlist = iDAL.ExecuteQuery<Entity>("usp_GetEntity", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetEntity?.Invoke(ref Retlist, ref getEntityReq);
            return Retlist;
        }

        #endregion

        #region GetCurrentContainerFileRelationship

        public List<CurrentContainerFileRelationship> GetCurrentContainerFileRelationship(
            GetCurrentContainerFileRelationshipReq getCurrentContainerFileRelationshipReq)
        {
            DAL.DAL iDAL = new();
            List<CurrentContainerFileRelationship> Retlist = [];
            OnPreEventGetCurrentContainerFileRelationship?.Invoke(ref getCurrentContainerFileRelationshipReq);

            DynamicParameters param = new();
            param.Add("Ids", ConvertIntListToString(getCurrentContainerFileRelationshipReq.Ids));

            Retlist = iDAL.ExecuteQuery<CurrentContainerFileRelationship>("usp_GetCurrentContainerFileRelationship",
                param, CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetCurrentContainerFileRelationship?.Invoke(ref Retlist,
                ref getCurrentContainerFileRelationshipReq);
            return Retlist;
        }

        #endregion

        #region GetCurrentFileStatusByFileId

        public List<FileStatus> GetCurrentFileStatusByFileId(
            GetCurrentFileStatusByFileIdReq getCurrentFileStatusByFileIdReq)
        {
            DAL.DAL iDAL = new();
            List<FileStatus> Retlist = [];
            OnPreEventGetCurrentFileStatusByFileId?.Invoke(ref getCurrentFileStatusByFileIdReq);

            DynamicParameters param = new();
            param.Add("FileId", getCurrentFileStatusByFileIdReq.FileId);

            Retlist = iDAL.ExecuteQuery<FileStatus>("usp_GetCurrentFileStatusByFileId", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetCurrentFileStatusByFileId?.Invoke(ref Retlist, ref getCurrentFileStatusByFileIdReq);
            return Retlist;
        }

        #endregion

        #region GetCustomerByWhere

        public List<Customer> GetCustomerByWhere(GetCustomerByWhereReq getCustomerByWhereReq)
        {
            DAL.DAL iDAL = new();
            List<Customer> Retlist = [];
            DynamicParameters param = new();
            OnPreEventGetCustomerByWhere?.Invoke(ref getCustomerByWhereReq);
            param.Add("Id", getCustomerByWhereReq.Id, DbType.Int32, ParameterDirection.Input);
            param.Add("ShortName", getCustomerByWhereReq.ShortName, DbType.String, ParameterDirection.Input);
            param.Add("LegalId", getCustomerByWhereReq.LegalId, DbType.String, ParameterDirection.Input);
            param.Add("PhoneNumberString", getCustomerByWhereReq.PhoneNumberString, DbType.String,
                ParameterDirection.Input);

            Retlist = iDAL.ExecuteQuery<Customer>("usp_GetCustomerByWhere", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetCustomerByWhere?.Invoke(ref Retlist, ref getCustomerByWhereReq);
            return Retlist;
        }

        #endregion

        #region GetArchivedFile

        public List<ArchivedFile> GetArchivedFile(GetArchivedFileReq getArchivedFileReq)
        {
            DAL.DAL iDAL = new();
            List<ArchivedFile> Retlist = [];
            OnPreEventGetArchivedFile?.Invoke(ref getArchivedFileReq);

            DynamicParameters param = new();
            param.Add("Ids", ConvertIntListToString(getArchivedFileReq.Ids));

            Retlist = iDAL.ExecuteQuery<ArchivedFile>("usp_GetFile", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetArchivedFile?.Invoke(ref Retlist, ref getArchivedFileReq);
            return Retlist;
        }

        #endregion

        #region GetFileName

        public List<FileName> GetFileName(GetFileNameReq getFileNameReq)
        {
            DAL.DAL iDAL = new();
            List<FileName> Retlist = [];
            OnPreEventGetFileName?.Invoke(ref getFileNameReq);

            Retlist = iDAL.ExecuteQuery<FileName>("usp_GetFileName", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetFileName?.Invoke(ref Retlist, ref getFileNameReq);
            return Retlist;
        }

        #endregion

        #region GetFilesByCustomerId

        public List<ArchivedFile> GetFilesByCustomerId(GetFilesByCustomerIdReq getFilesByCustomerIdReq)
        {
            DAL.DAL iDAL = new();
            List<ArchivedFile> Retlist = [];
            OnPreEventGetFilesByCustomerId?.Invoke(ref getFilesByCustomerIdReq);

            DynamicParameters param = new();
            param.Add("CustomerId", getFilesByCustomerIdReq.CustomerId);

            Retlist = iDAL.ExecuteQuery<ArchivedFile>("usp_GetFilesByCustomerId", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetFilesByCustomerId?.Invoke(ref Retlist, ref getFilesByCustomerIdReq);
            return Retlist;
        }

        #endregion

        #region GetFileStatus

        public List<FileStatus> GetFileStatus(GetFileStatusReq getFileStatusReq)
        {
            DAL.DAL iDAL = new();
            List<FileStatus> Retlist = [];
            OnPreEventGetFileStatus?.Invoke(ref getFileStatusReq);

            DynamicParameters param = new();
            param.Add("Ids", ConvertIntListToString(getFileStatusReq.Ids));

            Retlist = iDAL.ExecuteQuery<FileStatus>("usp_GetFileStatus", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetFileStatus?.Invoke(ref Retlist, ref getFileStatusReq);
            return Retlist;
        }

        #endregion

        #region GetFileType

        public List<FileType> GetFileType(GetFileTypeReq getFileTypeReq)
        {
            DAL.DAL iDAL = new();
            List<FileType> Retlist = [];
            OnPreEventGetFileType?.Invoke(ref getFileTypeReq);

            DynamicParameters param = new();
            param.Add("Entities", getFileTypeReq.BaseReq.CurrentEntity);

            Retlist = iDAL.ExecuteQuery<FileType>("usp_GetFileType", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetFileType?.Invoke(ref Retlist, ref getFileTypeReq);
            return Retlist;
        }

        #endregion

        #region GetAllFileType

        public List<FileType> GetAllFileType(GetAllFileTypeReq getAllFileTypeReq)
        {
            DAL.DAL iDAL = new();
            List<FileType> Retlist = [];
            OnPreEventGetAllFileType?.Invoke(ref getAllFileTypeReq);

            Retlist = iDAL.ExecuteQuery<FileType>("usp_GetAllFileType", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetAllFileType?.Invoke(ref Retlist, ref getAllFileTypeReq);
            return Retlist;
        }

        #endregion

        #region GetStatus

        public List<Status> GetStatus(GetStatusReq getStatusReq)
        {
            DAL.DAL iDAL = new();
            List<Status> Retlist = [];
            OnPreEventGetStatus?.Invoke(ref getStatusReq);

            Retlist = iDAL.ExecuteQuery<Status>("usp_GetStatus", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetStatus?.Invoke(ref Retlist, ref getStatusReq);
            return Retlist;
        }

        #endregion

        #region GetUserInteraction

        public List<UserInteraction> GetUserInteraction(GetUserInteractionReq getUserInteractionReq)
        {
            DAL.DAL iDAL = new();
            List<UserInteraction> Retlist = [];
            OnPreEventGetUserInteraction?.Invoke(ref getUserInteractionReq);

            DynamicParameters param = new();
            param.Add("Ids", ConvertIntListToString(getUserInteractionReq.Ids));

            Retlist = iDAL.ExecuteQuery<UserInteraction>("usp_GetUserInteraction", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetUserInteraction?.Invoke(ref Retlist, ref getUserInteractionReq);
            return Retlist;
        }

        #endregion

        #region GetUsers

        public List<Users> GetUsers(GetUsersReq getUsersReq)
        {
            DAL.DAL iDAL = new();
            List<Users> Retlist = [];
            OnPreEventGetUsers?.Invoke(ref getUsersReq);

            Retlist = iDAL.ExecuteQuery<Users>("usp_GetUsers", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetUsers?.Invoke(ref Retlist, ref getUsersReq);
            return Retlist;
        }

        #endregion

        #region GetWarehouse

        public List<Warehouse> GetWarehouse(GetWarehouseReq getWarehouseReq)
        {
            DAL.DAL iDAL = new();
            List<Warehouse> Retlist = [];
            OnPreEventGetWarehouse?.Invoke(ref getWarehouseReq);

            DynamicParameters param = new();
            param.Add("Codes", String.Join(",", getWarehouseReq.Codes));

            Retlist = iDAL.ExecuteQuery<Warehouse>("usp_GetWarehouse", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetWarehouse?.Invoke(ref Retlist, ref getWarehouseReq);
            return Retlist;
        }

        #endregion

        #region UpdateConfiguration

        public ArchiveConfiguration UpdateConfiguration(UpdateConfigurationReq updateConfigurationReq)
        {
            DAL.DAL iDAL = new();

            ArchiveConfiguration Ret = new();

            OnPreEventUpdateConfiguration?.Invoke(ref updateConfigurationReq);

            DynamicParameters param = new();

            param.Add("Id", updateConfigurationReq.Id);
            param.Add("SettingName", updateConfigurationReq.SettingName);
            param.Add("SettingValue", updateConfigurationReq.SettingValue);
            param.Add("IsActive", updateConfigurationReq.IsActive);
            param.Add("SettingDescription", updateConfigurationReq.SettingDescription);
            param.Add("User", updateConfigurationReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<ArchiveConfiguration>("usp_UpdateConfiguration", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateConfiguration?.Invoke(ref Ret, ref updateConfigurationReq);

            return Ret;
        }

        #endregion

        #region Download PDF enhanced method
        public String DownloadPDF(DownloadPDFReq downloadPDFReq)
        {
            OnPreEventDownloadPDF?.Invoke(ref downloadPDFReq);

            // Auto-detect document type if not provided
            if (downloadPDFReq.DocumentType == null)
            {
                downloadPDFReq.DocumentType = DetectDocumentType(downloadPDFReq.ContainerID);
            }

            String data = JsonConvert.SerializeObject(downloadPDFReq);
            HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
            HttpClient client = new();
            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ??
                                    throw new SGBLInternalServerException(
                                        "PDF Service not initialized please Contact Support");

            Task<HttpResponseMessage>
                Request = client.PostAsync($"{PDFRequestBase}RedownloadDocPDFForArchive", content);

            Request.Wait();
            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
            responseString.Wait();

            String Ret = responseString.Result;
            OnPostEventDownloadPDF?.Invoke(ref Ret, ref downloadPDFReq);

            return Ret;
        }
        #endregion

        private ArchivingDocumentType DetectDocumentType(String containerCode)
        {
            try
            {
                DAL.DAL iDAL = new();
                DynamicParameters param = new();
                param.Add("ContainerCode", containerCode);

                String command = ConfigurationManager.AppSettings["Get_Container_Data_For_PDF_Generation_SP"] ??
                                "usp_GetContainerDataForPDFGeneration";

                var containerData = iDAL.ExecuteQuery<dynamic>(command, param, CommandType.StoredProcedure,
                    CommandDirection.Select).FirstOrDefault();

                if (containerData == null)
                {
                    throw new SGBLInternalServerException($"Container {containerCode} not found");
                }

                String documentType = containerData.DocumentType;

                return documentType switch
                {
                    "CUSTOMER" => ArchivingDocumentType.CUSTOMER_PDF,
                    "BRANCH" => ArchivingDocumentType.BRANCH_PDF,
                    "ENTITY" => ArchivingDocumentType.ENTITY_PDF,
                    _ => ArchivingDocumentType.ENTITY_PDF // Default
                };
            }
            catch (Exception ex)
            {
                throw new SGBLInternalServerException($"Failed to detect document type for container {containerCode}", ex);
            }
        }
        private String GetActiveEntity(String entityCode)
        {
            try
            {
                DAL.DAL iDAL = new();
                DynamicParameters param = new();
                param.Add("EntityCode", entityCode);

                String command = ConfigurationManager.AppSettings["Get_Entity_By_Code_SP"] ??
                                "usp_GetEntityByCode";

                var entity = iDAL.ExecuteQuery<dynamic>(command, param, CommandType.StoredProcedure,
                    CommandDirection.Select).FirstOrDefault();

                return entity?.Description ?? entityCode;
            }
            catch
            {
                return entityCode;
            }
        }

        #region GetActiveEntity OLD ONE
        //public String GetActiveEntity(String EntityList)
        //{
        //    DAL.DAL iDAL = new();
        //    String? Ret = String.Empty;

        //    DynamicParameters param = new();

        //    param.Add("HoldingEntity", EntityList, DbType.String);

        //    Ret = iDAL.ExecuteQuery<String>("sp_GetHoldingEntity", param, CommandType.StoredProcedure,
        //        CommandDirection.Select).FirstOrDefault();


        //    return Ret ?? String.Empty;
        //}
        #endregion

        #region downloadPDF

        //public String DownloadPDF(DownloadPDFReq downloadPDFReq)
        //{
        //    OnPreEventDownloadPDF?.Invoke(ref downloadPDFReq);

        //    String data = JsonConvert.SerializeObject(downloadPDFReq);
        //    HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
        //    HttpClient client = new();
        //    String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ??
        //                            throw new SGBLInternalServerException(
        //                                "PDF Service not initialized please Contact Support");

        //    Task<HttpResponseMessage>
        //        Request = client.PostAsync($"{PDFRequestBase}RedownloadDocPDFForArchive", content);

        //    Request.Wait();
        //    Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
        //    responseString.Wait();

        //    String Ret = responseString.Result;
        //    OnPostEventDownloadPDF?.Invoke(ref Ret, ref downloadPDFReq);

        //    return Ret;
        //}

        #endregion

        #region downloadDestroyedBoxPDF

        public String DownloadDestroyedBoxPDF(DownloadDestroyedBoxPDFReq downloadDestroyedBoxPDFReq)
        {
            OnPreEventDownloadDestroyedBoxPDF?.Invoke(ref downloadDestroyedBoxPDFReq);

            String data = JsonConvert.SerializeObject(downloadDestroyedBoxPDFReq);
            HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
            HttpClient client = new();
            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ??
                                    throw new SGBLInternalServerException(
                                        "PDF Service not initialized please Contact Support");

            Task<HttpResponseMessage>
                Request = client.PostAsync($"{PDFRequestBase}RedownloadDocPDFForArchive", content);

            Request.Wait();
            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
            responseString.Wait();
            String Ret = responseString.Result;
            OnPostEventDownloadDestroyedBoxPDF?.Invoke(ref Ret, ref downloadDestroyedBoxPDFReq);
            return Ret;
        }

        #endregion

        #region UpdateContainer

        public Container UpdateContainer(UpdateContainerReq updateContainerReq)
        {
            DAL.DAL iDAL = new();

            Container Ret = new();

            OnPreEventUpdateContainer?.Invoke(ref updateContainerReq);

            DynamicParameters param = new();

            param.Add("Id", updateContainerReq.Id);
            param.Add("Code", updateContainerReq.Code);
            param.Add("CompanyCode", updateContainerReq.BaseReq.CurrentBranch);
            param.Add("Entity", updateContainerReq.BaseReq.CurrentEntity);
            param.Add("CurrentLocation", updateContainerReq.CurrentLocation);
            param.Add("StatusCode", updateContainerReq.StatusCode);
            param.Add("User", updateContainerReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Container>("usp_UpdateContainer", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateContainer?.Invoke(ref Ret, ref updateContainerReq);

            return Ret;
        }

        #endregion

        #region GetNextSequence

        public String GetNextSequence(BaseRequest baseReq)
        {
            DAL.DAL iDAL = new();

            DynamicParameters param = new();
            String Ret = String.Empty;
            param.Add("Entity", baseReq.CurrentEntity);
            param.Add("Branch", baseReq.CurrentBranch);
            param.Add("NextSequence", Ret, dbType: DbType.String, direction: ParameterDirection.Output);

            _ = iDAL.ExecuteQuery<String>("sp_GetNextSequence", param, CommandType.StoredProcedure,
                CommandDirection.Update);

            Ret = param.Get<String>("NextSequence");

            if (Ret == null)
            {
                Ret = String.Empty;
            }

            return Ret;
        }

        #endregion

        #region UpdateContainerStatus

        public ContainerStatus UpdateContainerStatus(UpdateContainerStatusReq updateContainerStatusReq)
        {
            DAL.DAL iDAL = new();

            ContainerStatus Ret = new();

            OnPreEventUpdateContainerStatus?.Invoke(ref updateContainerStatusReq);

            DynamicParameters param = new();

            param.Add("Id", updateContainerStatusReq.Id);
            param.Add("ContainerId", updateContainerStatusReq.ContainerId);
            param.Add("User", updateContainerStatusReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<ContainerStatus>("usp_UpdateContainerStatus", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateContainerStatus?.Invoke(ref Ret, ref updateContainerStatusReq);

            return Ret;
        }

        #endregion

        #region UpdateContainerType

        public ContainerType UpdateContainerType(UpdateContainerTypeReq updateContainerTypeReq)
        {
            DAL.DAL iDAL = new();

            ContainerType Ret = new();

            OnPreEventUpdateContainerType?.Invoke(ref updateContainerTypeReq);

            DynamicParameters param = new();

            param.Add("Id", updateContainerTypeReq.Id);
            param.Add("Code", updateContainerTypeReq.Code);
            param.Add("Description", updateContainerTypeReq.Description);
            param.Add("Category", updateContainerTypeReq.Category);
            param.Add("User", updateContainerTypeReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<ContainerType>("usp_UpdateContainerType", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateContainerType?.Invoke(ref Ret, ref updateContainerTypeReq);

            return Ret;
        }

        #endregion

        #region UpdateEntity

        public Entity UpdateEntity(UpdateEntityReq updateEntityReq)
        {
            DAL.DAL iDAL = new();

            Entity Ret = new();

            OnPreEventUpdateEntity?.Invoke(ref updateEntityReq);

            DynamicParameters param = new();

            param.Add("Id", updateEntityReq.Id);
            param.Add("Code", updateEntityReq.Code);
            param.Add("Description", updateEntityReq.Description);
            param.Add("HasFullAccess", updateEntityReq.HasFullAccess);
            param.Add("Category", updateEntityReq.Category);
            param.Add("User", updateEntityReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Entity>("usp_UpdateEntity", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateEntity?.Invoke(ref Ret, ref updateEntityReq);

            return Ret;
        }

        #endregion

        #region UpdateCurrentContainerFileRelationship

        public CurrentContainerFileRelationship UpdateCurrentContainerFileRelationship(
            UpdateCurrentContainerFileRelationshipReq updateCurrentContainerFileRelationshipReq)
        {
            DAL.DAL iDAL = new();

            CurrentContainerFileRelationship Ret = new();

            OnPreEventUpdateCurrentContainerFileRelationship?.Invoke(ref updateCurrentContainerFileRelationshipReq);

            DynamicParameters param = new();

            param.Add("Id", updateCurrentContainerFileRelationshipReq.Id);
            param.Add("FileId", updateCurrentContainerFileRelationshipReq.FileId);
            param.Add("ContainerId", updateCurrentContainerFileRelationshipReq.ContainerId);
            param.Add("User", updateCurrentContainerFileRelationshipReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<CurrentContainerFileRelationship>("usp_UpdateCurrentContainerFileRelationship",
                param, CommandType.StoredProcedure, CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateCurrentContainerFileRelationship?.Invoke(ref Ret,
                ref updateCurrentContainerFileRelationshipReq);

            return Ret;
        }

        #endregion

        #region UpdateFile

        public ArchivedFile UpdateFile(UpdateFileReq updateFileReq)
        {
            DAL.DAL iDAL = new();

            ArchivedFile Ret = new();

            OnPreEventUpdateFile?.Invoke(ref updateFileReq);

            DynamicParameters param = new();

            param.Add("Id", updateFileReq.Id);
            param.Add("CustomerId", updateFileReq.CustomerId);
            param.Add("Name", updateFileReq.Name);
            param.Add("FileTypeCode", updateFileReq.FileTypeCode);
            param.Add("CompanyCode", updateFileReq.BaseReq.CurrentBranch);
            param.Add("FromDate", updateFileReq.FromDate);
            param.Add("ToDate", updateFileReq.ToDate);
            param.Add("AdditionalInfo", updateFileReq.AdditionalInfo);
            param.Add("StatusCode", updateFileReq.StatusCode);
            param.Add("User", updateFileReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<ArchivedFile>("usp_UpdateFile", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateFile?.Invoke(ref Ret, ref updateFileReq);

            return Ret;
        }

        #endregion

        #region UpdateFileName

        public FileName UpdateFileName(UpdateFileNameReq updateFileNameReq)
        {
            DAL.DAL iDAL = new();

            FileName Ret = new();

            OnPreEventUpdateFileName?.Invoke(ref updateFileNameReq);

            DynamicParameters param = new();

            param.Add("Id", updateFileNameReq.Id);
            param.Add("Code", updateFileNameReq.Code);
            param.Add("Description", updateFileNameReq.Description);
            param.Add("FileType", updateFileNameReq.FileType);
            param.Add("User", updateFileNameReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<FileName>("usp_UpdateFileName", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateFileName?.Invoke(ref Ret, ref updateFileNameReq);

            return Ret;
        }

        #endregion

        #region UpdateFileStatus

        public FileStatus UpdateFileStatus(UpdateFileStatusReq updateFileStatusReq)
        {
            DAL.DAL iDAL = new();

            FileStatus Ret = new();

            OnPreEventUpdateFileStatus?.Invoke(ref updateFileStatusReq);

            DynamicParameters param = new();

            param.Add("Id", updateFileStatusReq.Id);
            param.Add("FileId", updateFileStatusReq.FileId);
            param.Add("StatusCode", updateFileStatusReq.StatusCode);
            param.Add("HoldingEntityCode", updateFileStatusReq.HoldingEntityCode);
            param.Add("IsCurrentStatus", updateFileStatusReq.isCurrentStatus);
            param.Add("User", updateFileStatusReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<FileStatus>("usp_UpdateFileStatus", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateFileStatus?.Invoke(ref Ret, ref updateFileStatusReq);

            return Ret;
        }

        #endregion

        #region UpdateFileType

        public FileType UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
        {
            DAL.DAL iDAL = new();

            FileType Ret = new();

            OnPreEventUpdateFileType?.Invoke(ref updateFileTypeReq);

            DynamicParameters param = new();

            param.Add("Id", updateFileTypeReq.Id);
            param.Add("Code", updateFileTypeReq.Code);
            param.Add("Entity", updateFileTypeReq.Entity);
            param.Add("Description", updateFileTypeReq.Description);
            param.Add("Category", updateFileTypeReq.Category);
            param.Add("HasDate", updateFileTypeReq.HasDate);
            param.Add("IsCustomer", updateFileTypeReq.IsCustomer);
            param.Add("ArchivingPeriod", updateFileTypeReq.ArchivingPeriod);
            param.Add("User", updateFileTypeReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<FileType>("usp_UpdateFileType", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateFileType?.Invoke(ref Ret, ref updateFileTypeReq);

            return Ret;
        }

        #endregion

        #region UpdateStatus

        public Status UpdateStatus(UpdateStatusReq updateStatusReq)
        {
            DAL.DAL iDAL = new();

            Status Ret = new();

            OnPreEventUpdateStatus?.Invoke(ref updateStatusReq);

            DynamicParameters param = new();

            param.Add("Id", updateStatusReq.Id);
            param.Add("Code", updateStatusReq.Code);
            param.Add("Description", updateStatusReq.Description);
            param.Add("Category", updateStatusReq.Category);
            param.Add("User", updateStatusReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Status>("usp_UpdateFileType", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateStatus?.Invoke(ref Ret, ref updateStatusReq);

            return Ret;
        }

        #endregion

        #region UpdateUserInteraction

        public UserInteraction UpdateUserInteraction(UpdateUserInteractionReq updateUserInteractionReq)
        {
            DAL.DAL iDAL = new();

            UserInteraction Ret = new();

            OnPreEventUpdateUserInteraction?.Invoke(ref updateUserInteractionReq);

            DynamicParameters param = new();

            param.Add("Id", updateUserInteractionReq.Id);
            param.Add("ContainerId", updateUserInteractionReq.ContainerId);
            param.Add("FromUser", updateUserInteractionReq.FromUser);
            param.Add("ToUser", updateUserInteractionReq.ToUser);
            param.Add("FromEntity", updateUserInteractionReq.FromEntity);
            param.Add("ToEntity", updateUserInteractionReq.ToEntity);
            param.Add("User", updateUserInteractionReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<UserInteraction>("usp_UpdateUserInteraction", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateUserInteraction?.Invoke(ref Ret, ref updateUserInteractionReq);

            return Ret;
        }

        #endregion

        #region UpdateUsers

        public Users UpdateUsers(UpdateUsersReq updateUsersReq)
        {
            DAL.DAL iDAL = new();

            Users Ret = new();

            OnPreEventUpdateUsers?.Invoke(ref updateUsersReq);

            DynamicParameters param = new();

            param.Add("Id", updateUsersReq.Id);
            param.Add("UserName", updateUsersReq.UserName);
            param.Add("Entity", updateUsersReq.Entity);
            param.Add("User", updateUsersReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Users>("usp_UpdateUsers", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateUsers?.Invoke(ref Ret, ref updateUsersReq);

            return Ret;
        }

        #endregion

        #region UpdateWarehouse

        public Warehouse UpdateWarehouse(UpdateWarehouseReq updateWarehouseReq)
        {
            DAL.DAL iDAL = new();

            Warehouse Ret = new();

            OnPreEventUpdateWarehouse?.Invoke(ref updateWarehouseReq);

            DynamicParameters param = new();

            param.Add("Code", updateWarehouseReq.Code);
            param.Add("WarehouseName", updateWarehouseReq.WarehouseName);
            param.Add("NameAddress", updateWarehouseReq.NameAddress);
            param.Add("Mnemonic", updateWarehouseReq.Mnemonic);
            param.Add("DisplayDescription", updateWarehouseReq.DisplayDescription);
            param.Add("User", updateWarehouseReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Warehouse>("usp_UpdateWarehouse", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateWarehouse?.Invoke(ref Ret, ref updateWarehouseReq);

            return Ret;
        }

        #endregion

        #region ReceiveContainer

        public Container ReceiveContainer(ReceiveContainerReq receiveContainerReq)
        {
            DAL.DAL iDAL = new();

            Container Ret = new();

            OnPreEventReceiveContainer?.Invoke(ref receiveContainerReq);

            DynamicParameters param = new();

            param.Add("ContainerId", receiveContainerReq.ContainerId);
            param.Add("User", receiveContainerReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Container>("usp_ReceiveContainer", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventReceiveContainer?.Invoke(ref Ret, ref receiveContainerReq);

            return Ret;
        }

        #endregion

        #region DeleteFile

        public Boolean DeleteFile(DeleteFileReq deleteFileReq)
        {
            DAL.DAL iDAL = new();

            Boolean Ret = false;

            OnPreEventDeleteFile?.Invoke(ref deleteFileReq);

            DynamicParameters param = new();


            param.Add("IsDeleted", DbType.Int32, direction: ParameterDirection.ReturnValue);
            param.Add("FileId", deleteFileReq.FileId);
            param.Add("User", deleteFileReq.BaseReq.CurrentUser);

            _ = iDAL.ExecuteQuery<Boolean>("usp_DeleteFile", param, CommandType.StoredProcedure,
                CommandDirection.Delete);

            Int32 isDeleted = param.Get<System.Int32>("IsDeleted");

            OnPostEventDeleteFile?.Invoke(ref deleteFileReq);

            if (isDeleted == 1)
            {
                Ret = true;
            }

            return Ret;
        }

        #endregion

        #region EditContainerStatus

        public Container EditContainerStatus(EditContainerStatusReq editContainerStatusReq)
        {
            DAL.DAL iDAL = new();

            Container Ret = new();

            OnPreEventEditContainerStatus?.Invoke(ref editContainerStatusReq);

            DynamicParameters param = new();

            param.Add("ContainerId", editContainerStatusReq.ContainerId);
            param.Add("StatusCode", editContainerStatusReq.StatusCode);
            param.Add("HoldingEntityCode", editContainerStatusReq.HoldingEntityCode);
            param.Add("User", editContainerStatusReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Container>("usp_EditContainerStatus", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventEditContainerStatus?.Invoke(ref Ret, ref editContainerStatusReq);

            return Ret;
        }

        #endregion

        #region EditFileStatus

        public ArchivedFile EditFileStatus(EditFileStatusReq editFileStatusReq)
        {
            DAL.DAL iDAL = new();

            ArchivedFile Ret = new();

            OnPreEventEditFileStatus?.Invoke(ref editFileStatusReq);

            DynamicParameters param = new();

            param.Add("FileId", editFileStatusReq.FileId);
            param.Add("StatusCode", editFileStatusReq.StatusCode);
            param.Add("HoldingEntityCode", editFileStatusReq.HoldingEntityCode);
            param.Add("User", editFileStatusReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<ArchivedFile>("usp_EditFileStatus", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventEditFileStatus?.Invoke(ref Ret, ref editFileStatusReq);

            return Ret;
        }

        #endregion

        #region RemoveFileFromContainer

        public Boolean RemoveFileFromContainer(RemoveFileFromContainerReq removeFileFromContainerReq)
        {
            DAL.DAL iDAL = new();

            Boolean Ret = new();

            OnPreEventRemoveFileFromContainer?.Invoke(ref removeFileFromContainerReq);

            DynamicParameters param = new();

            param.Add("FileId", removeFileFromContainerReq.FileId);
            param.Add("ContainerId", removeFileFromContainerReq.ContainerId);
            param.Add("User", removeFileFromContainerReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Boolean>("usp_RemoveFileFromContainer", param, CommandType.StoredProcedure,
                CommandDirection.Delete).FirstOrDefault()!;

            OnPostEventRemoveFileFromContainer?.Invoke(ref removeFileFromContainerReq);

            return Ret;
        }

        #endregion

        #region ValidateCustomer

        public String ValidateCustomer(ValidateCustomerReq validateCustomerReq)
        {
            DAL.DAL iDAL = new();
            String Ret = String.Empty;

            OnPreEventValidateCustomer?.Invoke(ref validateCustomerReq);

            DynamicParameters param = new();

            param.Add("Id", validateCustomerReq.Id, DbType.Int32, ParameterDirection.Input);
            param.Add("ContainerId", validateCustomerReq.ContainerId, DbType.Int32, ParameterDirection.Input);
            param.Add("ReturnedVal", dbType: DbType.String, direction: ParameterDirection.Output, size: 50);

            iDAL.ExecuteQuery<String>("usp_ValidateCustomer", param, CommandType.StoredProcedure,
                CommandDirection.Select);

            Ret = param.Get<String>("ReturnedVal");

            OnPostEventValidateCustomer?.Invoke(ref Ret, ref validateCustomerReq);

            return Ret;
        }

        #endregion

        #region GetWarehouseContainers

        public List<Container> GetWarehouseContainers(GetWarehouseContainersReq getWarehouseContainersReq)
        {
            DAL.DAL iDAL = new();
            List<Container> RetList = [];

            OnPreEventGetWarehouseContainers?.Invoke(ref getWarehouseContainersReq);

            DynamicParameters param = new();

            param.Add("FromDate", getWarehouseContainersReq.FromDate);
            param.Add("ToDate", getWarehouseContainersReq.ToDate);
            param.Add("Code", getWarehouseContainersReq.Code);
            param.Add("CompanyCode", getWarehouseContainersReq.CompanyCode);
            param.Add("StatusCode", getWarehouseContainersReq.StatusCode);

            RetList = iDAL.ExecuteQuery<Container>("usp_GetWarehouseContainers", param, CommandType.StoredProcedure,
                CommandDirection.Select);


            OnPostEventGetWarehouseContainers?.Invoke(ref RetList, ref getWarehouseContainersReq);

            return RetList;
        }

        #endregion

        #region GetContainerArchivingPeriod

        public List<ContainerArchivingPeriod> GetContainerArchivingPeriod(
            GetContainerArchivingPeriodReq getContainerArchivingPeriodReq)
        {
            DAL.DAL iDAL = new();
            List<ContainerArchivingPeriod> RetList = [];

            OnPreEventGetContainerArchivingPeriod?.Invoke(ref getContainerArchivingPeriodReq);

            DynamicParameters param = new();

            param.Add("ContainerIds", ConvertIntListToString(getContainerArchivingPeriodReq.ContainerIds));

            RetList = iDAL.ExecuteQuery<ContainerArchivingPeriod>("usp_GetContainerArchivingPeriod", param,
                CommandType.StoredProcedure, CommandDirection.Select);

            OnPostEventGetContainerArchivingPeriod?.Invoke(ref RetList, ref getContainerArchivingPeriodReq);

            return RetList;
        }

        #endregion

        #region GetSentContainers

        public List<Container> GetSentContainers(GetSentContainersReq getSentContainersReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetSentContainers?.Invoke(ref getSentContainersReq);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetSentContainers", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetSentContainers?.Invoke(ref Retlist, ref getSentContainersReq);
            return Retlist;
        }

        #endregion

        #region Get File Details By File Type

        public FileTypeDetails? GetFileTypeDetailsByFileTypeCode(
            GetFileTypeDetailsByFileTypeCodeReq getFileTypeDetailsByFileTypeCodeReq)
        {
            DAL.DAL iDAL = new();
            FileTypeDetails? fileTypeArchivingPeriod = new();

            OnPreEventGetFileTypeDetailsByFileTypeCode?.Invoke(ref getFileTypeDetailsByFileTypeCodeReq);

            DynamicParameters param = new();
            param.Add("FileTypeCode", getFileTypeDetailsByFileTypeCodeReq.FileTypeCode, DbType.String);

            fileTypeArchivingPeriod = iDAL.ExecuteQuery<FileTypeDetails>("sp_GetFileTypeDetailsByFileType", param,
                CommandType.StoredProcedure, CommandDirection.Select).FirstOrDefault();

            OnPostGetFileTypeDetailsByFileTypeCode?.Invoke(ref fileTypeArchivingPeriod,
                ref getFileTypeDetailsByFileTypeCodeReq);
            return fileTypeArchivingPeriod;
        }

        #endregion

        #region Get Entity Container By Status

        public List<Container> GetEntityContainerByStatus(GetEntityContainerByStatusReq getEntityContainerByStatusReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Containers = [];
            DynamicParameters param = new();
            OnPreEventGetEntityContainerByStatus?.Invoke(ref getEntityContainerByStatusReq);

            param.Add("Status", getEntityContainerByStatusReq.Status, DbType.String);
            param.Add("Entity", getEntityContainerByStatusReq.BaseReq.CurrentBranch, DbType.String);

            Containers = iDAL.ExecuteQuery<Container>("usp_GetEntityContainerByStatus", param,
                CommandType.StoredProcedure, CommandDirection.Select);

            OnPostEventGetEntityContainerByStatus?.Invoke(ref Containers, ref getEntityContainerByStatusReq);

            return Containers;
        }

        #endregion

        #region Get RCA Container By Status

        public List<Container> GetRCAContainerByStatus(GetRCAContainerByStatusReq getRCAContainerByStatusReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Containers = [];
            DynamicParameters param = new();
            OnPreEventGetRCAContainerByStatus?.Invoke(ref getRCAContainerByStatusReq);

            //param.Add("Status", getRCAContainerByStatusReq.Status, DbType.String);
            param.Add("Entity", getRCAContainerByStatusReq.BaseReq.CurrentEntity, DbType.String);
            param.Add("CompanyCode", getRCAContainerByStatusReq.BaseReq.CurrentBranch, DbType.String);

            Containers = iDAL.ExecuteQuery<Container>("usp_GetRCAContainerByStatus", param, CommandType.StoredProcedure,
                CommandDirection.Select);

            OnPostEventGetRCAContainerByStatus?.Invoke(ref Containers, ref getRCAContainerByStatusReq);

            return Containers;
        }

        #endregion

        #region GetContainerForEditByEntity

        public List<Container> GetContainerForEditByEntity(GetContainerForEditByEntityReq getContainerForEditByEntity)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerForEditByEntity?.Invoke(ref getContainerForEditByEntity);

            DynamicParameters param = new();
            param.Add("Entities", getContainerForEditByEntity.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainerForEditByEntity", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetContainerForEditByEntity?.Invoke(ref Retlist, ref getContainerForEditByEntity);
            return Retlist;
        }

        #endregion

        #region GetContainerForEditByRCA

        public List<Container> GetContainerForEditByRCA(GetContainerForEditByRCAReq getContainerForEditByRCAReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerForEditByRCA?.Invoke(ref getContainerForEditByRCAReq);

            DynamicParameters param = new();
            param.Add("CompanyCodes", getContainerForEditByRCAReq.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainerForEditByRCA", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerForEditByRCA?.Invoke(ref Retlist, ref getContainerForEditByRCAReq);
            return Retlist;
        }

        #endregion

        #region AddFileToContainer

        public Container AddFileToContainer(AddFileToContainerReq addFileToContainerReq)
        {
            DAL.DAL iDAL = new();

            Container Ret = new();

            OnPreEventAddFileToContainer?.Invoke(ref addFileToContainerReq);

            DynamicParameters param = new();

            param.Add("ContainerId", addFileToContainerReq.ContainerId);
            param.Add("CustomerId", addFileToContainerReq.CustomerId);
            param.Add("FileTypeCode", addFileToContainerReq.FileTypeCode);
            param.Add("FromDate", addFileToContainerReq.FromDate);
            param.Add("ToDate", addFileToContainerReq.ToDate);
            param.Add("AdditionalInfo", addFileToContainerReq.AdditionalInfo);
            param.Add("CompanyCode", addFileToContainerReq.BaseReq.CurrentBranch);
            param.Add("User", addFileToContainerReq.BaseReq.CurrentUser);

            _ = iDAL.ExecuteQuery<ArchivedFile>("usp_AddFileToContainer", param, CommandType.StoredProcedure,
                CommandDirection.Update);

            OnPostEventAddFileToContainer?.Invoke(ref Ret, ref addFileToContainerReq);

            return Ret;
        }

        #endregion

        #region GetAllBranches

        public List<Company> GetAllBranches(GetAllBranchesReq getAllBranchesReq)
        {
            DAL.DAL iDAL = new();
            List<Company> Retlist = [];
            OnPreEventGetAllBranches?.Invoke(ref getAllBranchesReq);

            Retlist = iDAL.ExecuteQuery<Company>("usp_GetAllBranches", null, CommandType.StoredProcedure,
                CommandDirection.Select);

            OnPostEventGetAllBranches?.Invoke(ref Retlist, ref getAllBranchesReq);

            return Retlist;
        }

        #endregion

        #region GetAllSequences

        public List<Sequence> GetAllSequences(GetAllSequencesReq getAllSequencesReq)
        {
            DAL.DAL iDAL = new();
            List<Sequence> Retlist = [];
            OnPreEventGetAllSequences?.Invoke(ref getAllSequencesReq);

            Retlist = iDAL.ExecuteQuery<Sequence>("usp_GetAllSequences", null, CommandType.StoredProcedure,
                CommandDirection.Select);

            OnPostEventGetAllSequences?.Invoke(ref Retlist, ref getAllSequencesReq);

            return Retlist;
        }

        #endregion

        #region UpdateSequence

        public Sequence UpdateSequence(UpdateSequenceReq updateSequenceReq)
        {
            DAL.DAL iDAL = new();

            Sequence Ret = new();

            OnPreEventUpdateSequence?.Invoke(ref updateSequenceReq);

            DynamicParameters param = new();

            param.Add("SequenceId", updateSequenceReq.SequenceId);
            param.Add("Owner", updateSequenceReq.Owner);
            param.Add("Prefix", updateSequenceReq.Prefix);
            param.Add("LastIndex", updateSequenceReq.LastIndex);
            param.Add("Suffix", updateSequenceReq.Suffix);
            param.Add("IsActive", updateSequenceReq.IsActive);
            param.Add("User", updateSequenceReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<Sequence>("usp_UpdateSequence", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateSequence?.Invoke(ref Ret, ref updateSequenceReq);

            return Ret;
        }

        #endregion

        #region Initialize Container Archiving Date

        public void InitializeContainerArchivingDate(
            InitializeContainerArchivingDateReq initializeContainerArchivingDateReq)
        {
            DAL.DAL iDAL = new();
            OnPreEventInitializeContainerArchivingDate?.Invoke(ref initializeContainerArchivingDateReq);

            DynamicParameters param = new();
            param.Add("Id", initializeContainerArchivingDateReq.Id, DbType.Int32);

            iDAL.ExecuteQuery("sp_InitializeContainerArchivingDate", param, CommandType.StoredProcedure,
                CommandDirection.Update);

            OnPostEventInitializeContainerArchivingDate?.Invoke(ref initializeContainerArchivingDateReq);
        }

        #endregion

        #region GetContainersToBeDestroyed

        public List<Container> GetContainersToBeDestroyed(ContainersToBeDestroyedReq containersToBeDestroyedReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventContainersToBeDestroyed?.Invoke(ref containersToBeDestroyedReq);

            DynamicParameters param = new();

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetToBeDestroyedContainers", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventContainersToBeDestroyed?.Invoke(ref Retlist, ref containersToBeDestroyedReq);
            return Retlist;
        }

        #endregion

        #region DestroyContainers

        public List<Container> DestroyContainers(DestroyContainersReq destroyContainersReq)
        {
            DAL.DAL iDAL = new();

            List<Container> Retlist = [];

            OnPreEventDestroyContainers?.Invoke(ref destroyContainersReq);

            DynamicParameters param = new();

            param.Add("ContainerIds", destroyContainersReq.ContainerIds);
            param.Add("User", destroyContainersReq.BaseReq.CurrentUser);

            Retlist = iDAL.ExecuteQuery<Container>("usp_DestroyContainerList", param, CommandType.StoredProcedure,
                CommandDirection.Update);

            OnPostEventDestroyContainers?.Invoke(ref Retlist, ref destroyContainersReq);

            return Retlist;
        }

        #endregion

        #region GetActiveCompaniesOfUser

        public List<Company> GetActiveCompaniesOfUser(GetActiveCompaniesOfUserReq getActiveCompaniesOfUserReq)
        {
            DAL.DAL iDAL = new();
            List<Company> Retlist = [];
            OnPreEventGetActiveCompaniesOfUser?.Invoke(ref getActiveCompaniesOfUserReq);

            DynamicParameters param = new();
            param.Add("UserCompanies", getActiveCompaniesOfUserReq.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Company>("usp_GetActiveCompaniesOfUser", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetActiveCompaniesOfUser?.Invoke(ref Retlist, ref getActiveCompaniesOfUserReq);
            return Retlist;
        }

        #endregion

        #region GetEntityFileTypes

        public List<FileType> GetEntityFileTypes(GetEntityFileTypesReq getEntityFileTypesReq)
        {
            DAL.DAL iDAL = new();
            List<FileType> Retlist = [];
            OnPreEventGetEntityFileTypes?.Invoke(ref getEntityFileTypesReq);

            DynamicParameters param = new();
            param.Add("Code", getEntityFileTypesReq.Code);

            Retlist = iDAL.ExecuteQuery<FileType>("usp_GetEntityFileTypes", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetEntityFileTypes?.Invoke(ref Retlist, ref getEntityFileTypesReq);
            return Retlist;
        }

        #endregion

        #region GetIsEntityValidationPermittedForContainer

        public Boolean GetIsEntityValidationPermittedForContainer(
            GetIsEntityValidationPermittedForContainerReq getIsEntityValidationPermittedForContainerReq)
        {
            DAL.DAL iDAL = new();
            Boolean RetVal;
            OnPreEventGetIsEntityValidationPermittedForContainer?.Invoke(
                ref getIsEntityValidationPermittedForContainerReq);

            DynamicParameters param = new();
            param.Add("ContainerId", getIsEntityValidationPermittedForContainerReq.ContainerId);
            param.Add("User", getIsEntityValidationPermittedForContainerReq.BaseReq.CurrentUser);
            param.Add("ReturnedVal", dbType: DbType.Boolean, direction: ParameterDirection.Output);

            _ = iDAL.ExecuteQuery<Boolean>("usp_GetIsEntityValidationPermittedForContainer", param,
                CommandType.StoredProcedure, CommandDirection.Select);

            RetVal = param.Get<Boolean>("ReturnedVal");

            OnPostEventGetIsEntityValidationPermittedForContainer?.Invoke(ref RetVal,
                ref getIsEntityValidationPermittedForContainerReq);
            return RetVal;
        }

        #endregion

        #region GetEntityFilesByFileType

        public List<ArchivedFile> GetEntityFilesByFileType(GetEntityFilesByFileTypeReq getEntityFilesByFileType)
        {
            DAL.DAL iDAL = new();
            List<ArchivedFile> Retlist = [];
            OnPreEventGetEntityFilesByFileType?.Invoke(ref getEntityFilesByFileType);

            DynamicParameters param = new();
            param.Add("FileTypeCode", getEntityFilesByFileType.FileTypeCode);
            param.Add("Entities", getEntityFilesByFileType.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<ArchivedFile>("usp_GetEntityFilesByFileType", param,
                CommandType.StoredProcedure, CommandDirection.Select);
            OnPostEventGetEntityFilesByFileType?.Invoke(ref Retlist, ref getEntityFilesByFileType);
            return Retlist;
        }

        #endregion

        #region GetBranchFileType

        public List<FileType> GetBranchFileType(GetBranchFileTypeReq getBranchFileTypeReq)
        {
            DAL.DAL iDAL = new();
            List<FileType> Retlist = [];
            OnPreEventGetBranchFileType?.Invoke(ref getBranchFileTypeReq);

            Retlist = iDAL.ExecuteQuery<FileType>("usp_GetBranchFileType", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetBranchFileType?.Invoke(ref Retlist, ref getBranchFileTypeReq);
            return Retlist;
        }

        #endregion

        #region GetContainerToNotifyWarehouse RCA

        public List<Container> GetContainerToNotifyWarehouse(GetContainerToNotifyWarehouseReq getContainerToNotifyWarehouseReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerToNotifyWarehouse?.Invoke(ref getContainerToNotifyWarehouseReq);

            DynamicParameters param = new();
            param.Add("CompanyCodes", getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainer_Sent_TobeNotifiedbyRCA", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerToNotifyWarehouse?.Invoke(ref Retlist, ref getContainerToNotifyWarehouseReq);
            return Retlist;
        }

        #endregion

        #region GetContainerToNotifyWarehouse Entity
        public List<Container> GetContainerToNotifyWarehouseByEntity(GetContainerToNotifyWarehouseByEntityReq getContainerToNotifyWarehouseByEntityReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerToNotifyWarehouseByEntity?.Invoke(ref getContainerToNotifyWarehouseByEntityReq);

            DynamicParameters param = new();
            param.Add("Entities", getContainerToNotifyWarehouseByEntityReq.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainer_Sent_TobeNotifiedbyEntity", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerToNotifyWarehouseByEntity?.Invoke(ref Retlist, ref getContainerToNotifyWarehouseByEntityReq);
            return Retlist;
        }
        #endregion

        #region NotifyContainersByIds
        public void NotifyContainersByIds(NotifyContainersByIdsReq req)
        {
            String pdfApiKey = ConfigurationManager.AppSettings["PDFService"] ??
                        throw new SGBLInternalServerException(
                            "PDF Service not initialized please Contact Support");

            DAL.DAL iDAL = new();

            DynamicParameters param = new DynamicParameters();
            param.Add("P__ContainerIds", req.ContainerIds);
            param.Add("P__User", req.BaseReq.CurrentUser);

            List<BoxesToBeDeliveredDto> retList = iDAL
                .ExecuteQuery<BoxesToBeDeliveredDto>(
                "usp_Get_Container_By_IdsList",
                param,
                CommandType.StoredProcedure,
                CommandDirection.Select);

            List<String> containerCompanyBooks = retList.Select((i) => i.CompanyCode).Distinct().ToList();

            ExportBoxesToBeDeliveredViewModel vm = new ExportBoxesToBeDeliveredViewModel()
            {
                Req = new ExportBoxesToBeDeliveredReq()
                {
                    User = req.BaseReq.CurrentUser!,
                    BranchList = string.Join(", ", containerCompanyBooks),
                    Entity = req.BaseReq.CurrentEntity!
                },
                BoxesToBeDeliveredList = retList
            };

            #region Send pdf via mail
            List<AttachmentDto>? attachments = new List<AttachmentDto>()
            {
                new AttachmentDto()
                {
                    FileMediaType = "application/pdf",
                    FileName = "ETSM_notification",
                    FileExtension = "pdf",
                    File = GenerateBranchBoxesToDelivered(vm)
                }
            };

            List<string> toList = (ConfigurationManager.AppSettings["sendTo"] ?? throw new ArgumentException("sendTo configuration is missing"))
                                   .Split(new[] { ',' }, StringSplitOptions.RemoveEmptyEntries)
                                   .Select(email => email.Trim())
                                   .ToList();

            SendEmail(
                "AlternaSysUser",
                Guid.NewGuid().ToString(),
                $"Archiving - {vm.Req.BranchList} | Boxes ready to be delivered",
                "Dears, \n\nPlease find attached file containing boxes ready to be delivered. \n\nRegards,",
                req.LoggedInUserEmail,
                 toList,
                attachments
                );

            #endregion
        }
        #endregion

        #region  SendEmail
        public void SendEmail(
            string userName,
            string guid,
            string subject,
            string body,
            string from,
            List<string> to,
            List<AttachmentDto>? attachments)
        {
            SendEmailRequest request = new SendEmailRequest()
            {
                Username = userName,
                CorrelationId = guid,
                EmailParameters = new EmailDto()
                {
                    EmailFrom = from,
                    EmailTo = to,
                    Subject = subject,
                    Body = body,
                    IsHtml = false,
                    IsZipped = false,
                    Priority = 0,
                    Attachments = attachments
                }
            };

            request.PublicKey = TokenGenerator
                .GeneratePublicKey(request.Username, request.CorrelationId, ConfigurationManager.AppSettings["EmailSalt"] ?? throw new ArgumentNullException("EmailSalt configuration is missing"));

            GlobalHttpClient.PostAsync<dynamic>(request, "http://uat-alterna-bn1:8011/api/Email/SendEmail").Wait();
        }
        #endregion

        #region ExportWarehouseContainers
        public String ExportWarehouseContainers(ExportWarehouseContainersReq req)
        {
            String pdfHexaString = String.Empty;
            try
            {
                List<ExportWareouseContainersDto> containerListPdf = [];

                foreach (Container container in req.ContainerList)
                {
                    containerListPdf.Add(new()
                    {
                        ContainerCode = container.Code,
                        Branch = container.CompanyCode,
                        StatusCode = container.StatusCode,
                        ArchivingDate = (container.ArchivingDate.HasValue) ? container.ArchivingDate.Value.ToString("dd/MM/yyyy") : String.Empty,
                        DestructionDate = (container.DestructionDate.HasValue) ? container.DestructionDate.Value.ToString("dd/MM/yyyy") : String.Empty,
                        ArchivingPeriod = container.ArchivingPeriod,
                        SentBy = container.SentBy,
                    });
                }

                ExportWarehouseContainersPdfReq pdfReq = new ExportWarehouseContainersPdfReq()
                {
                    User = req.BaseReq.CurrentUser ?? "N/A",
                    ContainerList = containerListPdf,
                    BranchList = req.BaseReq.CurrentBranch ?? "N/A",
                    Entity = req.BaseReq.CurrentEntity ?? "N/A"
                };

                String data = JsonConvert.SerializeObject(pdfReq);
                HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
                HttpClient client = new();
                String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ?? throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}ExportWarehouseContainers", content);

                Request.Wait();

                Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                responseString.Wait();

                pdfHexaString = responseString.Result;
            }
            catch (Exception ex)
            {
                throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
            }

            return pdfHexaString;
        }
        #endregion
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

            oldBoxes.ForEach(bx =>
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
                dr["BoxSentDate"] = bx.BoxSentDate;
                dr["LastIndex"] = bx.LastIndex;

                dt.Rows.Add(dr);

                rowNbr++;
            });

            return dt;
        }

        #region BackfillResult for old Boxes
        public BackfillResult BackfillMissingPDFsForLegacyContainers(string currentUser, DateTime? fromDate = null, DateTime? toDate = null)
        {
            BackfillResult result = new();

            try
            {
                DAL.DAL _dal = new();

                // Get all containers without PDF using stored procedure
                DynamicParameters parameters = new DynamicParameters();
                parameters.Add("FromDate", fromDate, DbType.DateTime, ParameterDirection.Input);
                parameters.Add("ToDate", toDate, DbType.DateTime, ParameterDirection.Input);

                string command = ConfigurationManager.AppSettings["Get_Containers_Without_PDF_SP"] ??
                               "usp_GetContainersWithoutPDF";

                var containersWithoutPDF = _dal.ExecuteQuery<BackfillContainerInfo>(
                    command,
                    parameters,
                            CommandType.StoredProcedure,
                            CommandDirection.Select
                );

                result.TotalContainers = containersWithoutPDF.Count();

                foreach (var container in containersWithoutPDF)
                {
                    try
                    {
                        // Get the actual user who sent the container using stored procedure
                        string sentByUser = GetContainerSentByUser(container.Id);

                        // Get container files using stored procedure
                        DynamicParameters fileParam = new();
                        fileParam.Add("ContainerId", container.Id);

                        string fileCommand = ConfigurationManager.AppSettings["Get_Container_Files_For_Backfill_SP"] ??
                                           "usp_GetContainerFilesForBackfill";

                        var files = _dal.ExecuteQuery<BackfillFileInfo>(
                            fileCommand,
                            fileParam,
                            CommandType.StoredProcedure,
                            CommandDirection.Select
                        ).ToList();

                        if (files.Count > 0)
                        {
                            var firstFile = files.First();

                            bool unlimited = firstFile.ArchivingPeriod == -1;
                            DateTime archivePeriod = container.ArchivingDate ?? DateTime.Now;
                            archivePeriod = archivePeriod.AddYears(firstFile.ArchivingPeriod);

                            string entity = GetActiveEntity(container.Entity);
                            string destructionDate = unlimited ? "Unlimited" : $"{archivePeriod:dd/MM/yyyy}";
                            string creationDate = container.ArchivingDate.HasValue
                                ? $"{container.ArchivingDate.Value:dd/MM/yyyy}"
                                : $"{DateTime.Now:dd/MM/yyyy}";

                            // Determine document type and generate PDF
                            if (firstFile.CustomerId != null)
                            {
                                GenerateCustomerPDFForLegacyContainer(files, container.Code, entity, sentByUser, destructionDate, creationDate, _dal); ;
                            }
                            else if (firstFile.CompanyCode.StartsWith("LB"))
                            {
                                GenerateBranchPDFForLegacyContainer(files, container.Code, entity, sentByUser, destructionDate, creationDate);
                            }
                            else if (firstFile.CompanyCode.StartsWith("ET"))
                            {
                                GenerateEntityPDFForLegacyContainer(files, container.Code, firstFile.CompanyCode, sentByUser, destructionDate, creationDate);
                            }

                            result.SuccessCount++;
                        }
                        else
                        {
                            result.FailureCount++;
                            result.FailedContainers.Add($"{container.Code}: No files found");
                        }
                    }
                    catch (Exception ex)
                    {
                        result.FailureCount++;
                        result.FailedContainers.Add($"{container.Code}: {ex.Message}");
                        // Log the error
                        LogError($"Failed to generate PDF for container {container.Code}", new CorrelationInfo(), ex);
                    }
                }
            }
            catch (Exception ex)
            {
                throw new SGBLInternalServerException("Backfill process failed", ex);
            }

            return result;
        }

        private string GetContainerSentByUser(int containerId)
        {
            try
            {

                DAL.DAL _dal = new();

                DynamicParameters parameters = new DynamicParameters();
                parameters.Add("ContainerId", containerId);

                string command = ConfigurationManager.AppSettings["Get_Container_Sent_By_User_SP"] ??
                                "usp_GetContainerSentByUser";

                var result = _dal.ExecuteQuery<dynamic>(
                    command,
                    parameters,
                CommandType.StoredProcedure,
                CommandDirection.Select
                ).FirstOrDefault();

                return result?.SentByUser ?? "System";
            }
            catch
            {
                return "System";
            }
        }

        #region Generate PDF for for Legacy Containers [Entity - Branches - Customers]

        #endregion
        private void GenerateEntityPDFForLegacyContainer(List<BackfillFileInfo> files, string containerCode,
            string entity, string user, string destructionDate, string creationDate)
        {
            EntityDocRequest entityDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = entity,
                User = user,
                EntityFiles = new(),
                CreationDate = creationDate
            };

            foreach (BackfillFileInfo item in files)
            {
                entityDocRequest.EntityFiles.Add(new()
                {
                    DocumentType = item.FileName,
                    DocumentDescription = item.AdditionalInfo ?? string.Empty
                });
            }

            CallPDFServiceAndStore(entityDocRequest, "GenerateEntityDocPDFForArchive");
        }

        private void GenerateBranchPDFForLegacyContainer(List<BackfillFileInfo> files, string containerCode,
            string entity, string user, string destructionDate, string creationDate)
        {
            BranchDocRequest branchDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = entity,
                User = user,
                BranchFiles = new(),
                CreationDate = creationDate
            };

            foreach (BackfillFileInfo item in files)
            {
                branchDocRequest.BranchFiles.Add(new()
                {
                    DocumentType = item.FileName,
                    FromDate = item.FromDate.HasValue ? $"{item.FromDate.Value:dd-MM-yyyy}" : "",
                    ToDate = item.ToDate.HasValue ? $"{item.ToDate.Value:dd-MM-yyyy}" : ""
                });
            }

            CallPDFServiceAndStore(branchDocRequest, "GenerateBranchDocPDFForArchive");
        }

        private void GenerateCustomerPDFForLegacyContainer(List<BackfillFileInfo> files, string containerCode,
            string entity, string user, string destructionDate, string creationDate, DAL.DAL iDAL)
        {
            CustomerDocRequest customerDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = entity,
                User = user,
                CustomerFiles = new(),
                CreationDate = creationDate
            };

            // Get customer files grouped by document type using stored procedure
            DynamicParameters param = new();
            param.Add("ContainerCode", containerCode);

            string command = ConfigurationManager.AppSettings["Get_Customer_Files_By_Container_SP"] ??
                           "usp_GetCustomerFilesByContainer";

            var customerFiles = iDAL.ExecuteQuery<dynamic>(
                command,
                param,
                CommandType.StoredProcedure,
                CommandDirection.Select
            ).ToList();

            // Group customer IDs by document type
            Dictionary<string, List<string>> fileDict = new();
            foreach (var file in customerFiles)
            {
                string docType = file.DocumentType;
                string customerId = file.CustomerIdString ?? "";

                if (!fileDict.ContainsKey(docType))
                {
                    fileDict.Add(docType, new List<string> { customerId });
                }
                else if (!fileDict[docType].Contains(customerId))
                {
                    fileDict[docType].Add(customerId);
                }
            }

            foreach (KeyValuePair<string, List<string>> dictEntry in fileDict)
            {
                customerDocRequest.CustomerFiles.Add(new()
                {
                    DocumentType = dictEntry.Key,
                    Id = dictEntry.Value
                });
            }

            CallPDFServiceAndStore(customerDocRequest, "GenerateCustomerDocPDFForArchive");
        }

        private void CallPDFServiceAndStore(object request, string apiMethod)
        {
            try
            {
                string data = JsonConvert.SerializeObject(request);
                HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
                HttpClient client = new();
                string pdfRequestBase = ConfigurationManager.AppSettings["PDFService"]
                    ?? throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                Task<HttpResponseMessage> httpRequest = client.PostAsync($"{pdfRequestBase}{apiMethod}", content);
                httpRequest.Wait();

                Task<string> responseString = httpRequest.Result.Content.ReadAsStringAsync();
                responseString.Wait();

                if (string.IsNullOrEmpty(responseString.Result))
                {
                    throw new SGBLInternalServerException("PDF Service returned empty response");
                }
            }
            catch (Exception ex)
            {
                throw new SGBLInternalServerException($"PDF Creation Failed for {apiMethod}", ex);
            }
        }

        #endregion
    }
}
using System.Globalization;
using System.Net;
using System.Text.RegularExpressions;

using Alterna.Archive.Core.Global;
using Alterna.Archive.Core.Models;
using Alterna.Archive.Core.Models.TableModel;

using ClosedXML.Excel;

using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;

using static NLog.NLogUtil;

namespace Alterna.Archive.Core.Controllers
{
    public class ConfigurationController : BaseController
    {
        private readonly ExcelValidationService _validationService;
        public ConfigurationController(ExcelValidationService validationService)
        {
            _validationService = validationService;
        }
        public ActionResult GeneralSettings() => View();
        public ActionResult EntityManagement() => View();
        public ActionResult FileTypeManagement()
        {
            String session = GetSession("ArchiveData");
            FileTypeModel model = new();
            GetActiveCompaniesOfUserRes getActiveCompaniesOfUserRes = Common.ApiCall<GetActiveCompaniesOfUserRes>(new GetActiveCompaniesOfUserReq() { BaseReq = new BaseRequest(HttpContext, session, false) }, "GetActiveCompaniesOfUser");

            foreach (Company comp in getActiveCompaniesOfUserRes.Resp)
            {
                model.EntityList.Add(new SelectListItem { Text = $"{comp.Code} - {comp.Mnemonic}", Value = comp.Code });
            }
            return View(model);
        }

        public ActionResult ReDownloadPDF(ErrorModel em) => View(em);
        public ActionResult DownloadPDF(String boxReference)
        {
            DownloadPDFModel model = new();
            DownloadPDFRes downloadPDFRes = Common.ApiCall<DownloadPDFRes>(new DownloadPDFReq()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                ContainerID = boxReference
            }, "DownloadPDF");

            if (downloadPDFRes.Resp is null || downloadPDFRes.Resp.Length == 0)
            {
                return View("ReDownloadPDF", new ErrorModel() { ErrorMessage = "Incorrect Box Reference!" });
            }

            String PDF = downloadPDFRes.Resp! ?? String.Empty;

            if (PDF == String.Empty)
            {
                HttpContext.Session.SetString("CorrelationId", downloadPDFRes.WebResp.CorrelationId);
                HttpContext.Session.SetString("ErrorMessage", "Invalid PDF");

                throw new ErrorHandler(new ErrorModel() { ErrorCorrelationId = downloadPDFRes.WebResp.CorrelationId, ErrorMessage = "PDF Server Not Responding" });
            }

            Byte[] bytearray = new Byte[PDF.Length / 2];
            for (Int32 i = 0; i < PDF.Length; i += 2)
            {
                bytearray[i / 2] = Convert.ToByte(PDF.Substring(i, 2), 16);
            }

            String ModifiedRef = boxReference;
            Regex specialCharacters = new("""
                                            [<]|[>]|[:]|["]|[/]|[\\]|[|]|[?]|[*]
                                            """);
            ModifiedRef = specialCharacters.Replace(ModifiedRef, "_");

            FileContentResult fileContentResult = new(bytearray, "application/pdf")
            {
                FileDownloadName = $"{ModifiedRef}_{DateTime.Now:yyyy-MM-dd hh-mm-ss}.pdf"
            };

            return fileContentResult;
        }

        public ActionResult ReDownloadDestroyedBoxPDF(ErrorModel em) => View(em);
        public ActionResult DownloadDestroyedBoxPDF(String boxReference)
        {
            DownloadDestroyedBoxPDFModel model = new();
            DownloadDestroyedBoxPDFRes downloadDestroyedBoxPDFRes = Common.ApiCall<DownloadDestroyedBoxPDFRes>(new DownloadDestroyedBoxPDFReq()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                ContainerID = boxReference
            }, "DownloadDestroyedBoxPDF");

            if (downloadDestroyedBoxPDFRes.Resp is null || downloadDestroyedBoxPDFRes.Resp.Length == 0)
            {
                return View("ReDownloadDestroyedBoxPDF", new ErrorModel() { ErrorMessage = "Incorrect Box Reference!" });
            }

            String PDF = downloadDestroyedBoxPDFRes.Resp! ?? String.Empty;

            if (PDF == String.Empty)
            {
                HttpContext.Session.SetString("CorrelationId", downloadDestroyedBoxPDFRes.WebResp.CorrelationId);
                HttpContext.Session.SetString("ErrorMessage", "Invalid PDF");

                throw new ErrorHandler(new ErrorModel() { ErrorCorrelationId = downloadDestroyedBoxPDFRes.WebResp.CorrelationId, ErrorMessage = "PDF Server Not Responding" });
            }

            Byte[] bytearray = new Byte[PDF.Length / 2];
            for (Int32 i = 0; i < PDF.Length; i += 2)
            {
                bytearray[i / 2] = Convert.ToByte(PDF.Substring(i, 2), 16);
            }

            String ModifiedRef = boxReference;
            Regex specialCharacters = new("""
                                            [<]|[>]|[:]|["]|[/]|[\\]|[|]|[?]|[*]
                                            """);
            ModifiedRef = specialCharacters.Replace(ModifiedRef, "_");

            FileContentResult fileContentResult = new(bytearray, "application/pdf")
            {
                FileDownloadName = $"{ModifiedRef}_{DateTime.Now:yyyy-MM-dd hh-mm-ss}.pdf"
            };

            return fileContentResult;
        }

        public ActionResult BoxSequenceManagement()
        {
            SequenceModel model = new();

            GetAllBranchesRes getAllBranchesRes = new();
            GetEntityRes getEntityRes = new();

            getAllBranchesRes = Common.ApiCall<GetAllBranchesRes>(new GetAllBranchesReq() { BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")) }, "GetAllBranches");
            //getEntityRes = Common.ApiCall<GetEntityRes>(new GetEntityReq() { BaseReq = new BaseRequest(HttpContext,GetSession("ArchiveData")) }, "GetEntity");

            //foreach (Entity e in getEntityRes.Resp)
            //{
            //    model.OwnerList.Add(new SelectListItem { Text = e.Code, Value = e.Code });
            //}

            foreach (Company c in getAllBranchesRes.Resp)
            {
                model.OwnerList.Add(new SelectListItem { Text = c.Mnemonic, Value = c.Code });
            }

            return View(model);
        }

        public ActionResult Configuration()
        {
            ConfigurationModel model = new();
            GetAllConfigurationsRes resp = new();
            resp = Common.ApiCall<GetAllConfigurationsRes>(new GetAllConfigurationsReq() { BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")) }, "GetAllConfigurations");
            model.ConfigurationList = resp.Resp;
            return PartialView("_GeneralSettingsTable", model);
        }

        public ActionResult FileType(FileTypeModel fileTypeModel)
        {
            FileTypeModel model = new();
            GetFileTypeRes resp = new();
            resp = Common.ApiCall<GetFileTypeRes>(new GetEntityReq() { BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")) }, "GetAllFileType");
            model.FileTypeList = resp.Resp;
            foreach (FileType ft in model.FileTypeList)
            {
                ft.IsBranch = ft.Category.ToLower().Equals("branch");
            }
            model.EntityList = fileTypeModel.EntityList;
            return PartialView("_FileTypeManagementTable", model);
        }

        public Boolean UpdateFileType(Int32 ModelID, String code, String Entity, String Description, Boolean IsBranch, Boolean IsCustomer, String ArchivingPeriod)
        {
            if (!Int32.TryParse(ArchivingPeriod, out Int32 AP))
            {
                AP = -2;
            }

            if (Entity is null)
            {
                Entity = String.Empty;
            }

            UpdateFileTypeReq updateFileTypeReq = new()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                Id = ModelID,
                Code = code,
                Entity = Entity,
                Description = Description,
                IsCustomer = IsCustomer,
                HasDate = !IsCustomer,
                Category = (IsBranch == true) ? "Branch" : "Not Branch",
                ArchivingPeriod = AP
            };
            UpdateFileTypeRes resp = new();
            resp = Common.ApiCall<UpdateFileTypeRes>(updateFileTypeReq, "UpdateFileType");
            return true;
        }

        public ActionResult Entity()
        {
            EntityModel model = new();
            GetEntityRes resp = new();
            resp = Common.ApiCall<GetEntityRes>(new GetEntityReq() { BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")) }, "GetEntity");
            model.EntityList = resp.Resp;
            return PartialView("_EntityManagementTable", model);
        }

        public Boolean UpdateEntity(Int32 ModelID, String code, String description, Boolean hasFullAccess, String category)
        {
            UpdateEntityReq updateEntityReq = new()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                Id = ModelID,
                Code = code,
                Description = description,
                HasFullAccess = hasFullAccess,
                Category = category
            };
            UpdateEntityRes resp = new();
            resp = Common.ApiCall<UpdateEntityRes>(updateEntityReq, "UpdateEntity");
            return true;
        }

        public ActionResult Sequence(SequenceModel sequenceModel)
        {
            SequenceModel model = new();

            GetAllSequencesRes resp = new();

            resp = Common.ApiCall<GetAllSequencesRes>(new GetAllSequencesReq() { BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")) }, "GetAllSequences");

            model.SequenceList = resp.Resp;

            model.OwnerList = sequenceModel.OwnerList.Where(p => !model.SequenceList.Any(p2 => p2.Owner == p.Value)).ToList();

            return PartialView("_SequenceManagementTable", model);
        }

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

    #region Models

    public class ValidationResult
    {
        public List<ValidationError> Errors { get; set; } = new();
        public bool IsValid => !Errors.Any();

        public void AddError(string location, string message)
        {
            Errors.Add(new ValidationError { Location = location, Message = message });
        }
    }

    public class ValidationError
    {
        public string Location { get; set; } = string.Empty;
        public string Message { get; set; } = string.Empty;
    }

    public class ExcelValidationConfig
    {
        public int WorksheetIndex { get; set; } = 1; // ClosedXML uses 1-based indexing
        public int HeaderRow { get; set; } = 1;
        public int DataStartRow { get; set; } = 2;
        public bool CaseSensitiveHeaders { get; set; } = false;
        public int MinimumDataRows { get; set; } = 1;
        public List<ColumnConfig> ExpectedHeaders { get; set; } = new();
    }

    public class ColumnConfig
    {
        public string Name { get; set; }
        public CellDataType DataType { get; set; }
        public bool IsRequired { get; set; }
        public bool MustBeUnique { get; set; }

        // Text validations
        public int? MinLength { get; set; }
        public int? MaxLength { get; set; }
        public string RegexPattern { get; set; }

        // Numeric validations
        public decimal? MinValue { get; set; }
        public decimal? MaxValue { get; set; }
        public int? MaxDecimalPlaces { get; set; }

        // Date validations
        public DateTime? MinDate { get; set; }
        public DateTime? MaxDate { get; set; }
        public bool DisallowFutureDates { get; set; }
    }

    public enum CellDataType
    {
        Text,
        Number,
        Date,
        Boolean
        //Email,
        //Phone
    }

    #endregion

    #region Excel File Validation OLD CODE
    //private bool ValidateFiles(IFormFile? excelFile, out string message)
    //{
    //    message = string.Empty;

    //    if (excelFile == null || excelFile.Length == 0)
    //    {
    //        message = "Excel file is required.";

    //        return false;
    //    }

    //    string generalExtension = Path.GetExtension(excelFile.FileName).ToLower();

    //    if (generalExtension != ".xlsx")
    //    {
    //        message = "file must have a .xlsx extension.";

    //        return false;
    //    }

    //    return true;
    //}
    #endregion

}
namespace ALTERNA.ARCHIVING.BLL
{
    public partial class BLL
    {
        #region Base BLL
        public String CurrentUser { get; }
        #endregion
    }

    #region Configuration
    // This class was named ArchiveConfiguration because the name COnfiguration is ambiguous with the microsoft Configuration class
    public partial class ArchiveConfiguration
    {
        public Int32 Id { get; set; }
        public String SettingName { get; set; } = String.Empty;
        public String SettingValue { get; set; } = String.Empty;
        public Boolean IsActive { get; set; }
        public String? SettingDescription { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerType
    public partial class ContainerType
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String? Description { get; set; } = String.Empty;
        public String Category { get; set; } = String.Empty;




    }
    #endregion
    #region Entity
    public partial class Entity
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String? Description { get; set; } = String.Empty;
        public Boolean HasFullAccess { get; set; }
        public String Category { get; set; } = String.Empty;




    }
    #endregion
    #region FileName   

    public partial class FileName
    {
        public Int32 Id { set; get; }
        public String Code { get; set; } = String.Empty;
        public String? Description { get; set; } = String.Empty;
        public String FileType { get; set; } = String.Empty;




    }
    #endregion
    #region FileType
    public partial class FileType
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;
        public String? Description { get; set; }
        public String Category { get; set; } = String.Empty;
        public Boolean HasDate { get; set; }
        public Boolean IsCustomer { get; set; }
        public Int32 ArchivingPeriod { get; set; }




    }
    #endregion
    #region Status
    public partial class Status
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String? Description { get; set; }
        public String Category { get; set; } = String.Empty;




    }
    #endregion
    #region Users
    public partial class Users
    {
        public Int32 Id { get; set; }
        public String Username { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;




    }
    #endregion
    #region Company
    public partial class Company
    {
        public String Code { get; set; } = String.Empty;
        public String CompanyName { get; set; } = String.Empty;
        public String NameAddress { get; set; } = String.Empty;
        public String Mnemonic { get; set; } = String.Empty;
        public String? DisplayDescription { get; set; } = String.Empty;
        public Boolean IsBranch { get; set; }
        public Boolean IsActive { get; set; }



    }
    #endregion
    #region Container
    public partial class Container
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String CompanyCode { get; set; } = String.Empty;
        public String CompanyName { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;
        public String CurrentLocation { get; set; } = String.Empty;
        public String StatusCode { get; set; } = String.Empty;
        public DateTime? ArchivingDate { get; set; }
        public Boolean IsDeleted { get; set; }
        //public Boolean? IsNotified { get; set; }
        public String CreatedBy { get; set; } = String.Empty;
        public DateTime CreatedDate { get; set; }
        public String LastModifiedBy { get; set; } = String.Empty;
        public DateTime LastModifiedDate { get; set; }

    }
    #endregion
    #region ContainerDisposeDocumentation
    public partial class ContainerDisposeDocumentation
    {
        public Int64 Id { get; set; }
        public Int64 ContainerDisposeRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;





    }
    #endregion
    #region ContainerDisposeRequest
    public partial class ContainerDisposeRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerPickupDocumentation
    public partial class ContainerPickupDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 ContainerPickupRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerPickupRequest
    public partial class ContainerPickupRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerRentDocumentation
    public partial class ContainerRentDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 ContainerRentRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerRentRequest
    public partial class ContainerRentRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerReturnDocumentation
    public partial class ContainerReturnDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 ContainerReturnRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerReturnRequest
    public partial class ContainerReturnRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerStatus
    public partial class ContainerStatus
    {
        public Int64 Id { get; set; }
        public Int32 ContainerId { get; set; }
        public String StatusCode { get; set; } = String.Empty;
        public String HoldingEntityCode { get; set; } = String.Empty;
        public Boolean isCurrentStatus { get; set; }
        public String CreatedBy { get; set; } = String.Empty;
        public DateTime CreatedDate { get; set; }
    }
    #endregion
    #region CurrentContainerFileRelationship
    public partial class CurrentContainerFileRelationship
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region Customer
    public partial class Customer
    {
        public Int32 Id { get; set; }
        public String? CompanyBook { get; set; }
        public String? ShortName { get; set; }
        public String? GivenNames { get; set; }
        public String? FamilyName { get; set; }
        public String? FatherName { get; set; }
        public String? MoFirstName { get; set; }
        public String? MoLastName { get; set; }
        public String? LegalId { get; set; }
        public String? PCntryCode { get; set; }
        public String? PhoneAreaCode { get; set; }
        public String? PhoneNo { get; set; }
        public String? MCntryCode { get; set; }
        public String? LbmbAreaCode { get; set; }
        public String? LbmbMobilenb { get; set; }
        public String? AddCustType { get; set; }





    }
    #endregion
    #region ArchivedFile
    public partial class ArchivedFile
    {
        public Int32 Id { get; set; }
        public Int32? CustomerId { get; set; }
        //public String CustomerName { get; set; } = String.Empty;
        public String Name { get; set; } = String.Empty;
        public String CompanyCode { get; set; } = String.Empty;
        public String CompanyName { get; set; } = String.Empty;
        public String FileTypeCode { get; set; } = String.Empty;
        public String Status { get; set; } = String.Empty;
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? AdditionalInfo { get; set; }
        public Boolean isDeleted { get; set; }
        public DateTime CreatedDate { get; set; }
        public String CreatedBy { get; set; } = String.Empty;
    }
    #endregion
    #region FileClient
    public partial class FileClient
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }
        public Int32 ClientId { get; set; }




    }
    #endregion
    #region FileDisposeDocumentation
    public partial class FileDisposeDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 FileDisposeRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region FileDisposeRequest
    public partial class FileDisposeRequest
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }




    }
    #endregion
    #region FileRentDocumentation
    public partial class FileRentDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 FileRentRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region FileRentRequest
    public partial class FileRentRequest
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }




    }
    #endregion
    #region FileReturnDocumentation
    public partial class FileReturnDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 FileReturnRequestId { get; set; }
        public Byte[] Attachment { get; set; } = Array.Empty<Byte>();
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region FileReturnRequest
    public partial class FileReturnRequest
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }




    }
    #endregion
    #region FileStatus
    public partial class FileStatus
    {
        public Int64 Id { get; set; }
        public Int32 FileId { get; set; }
        public String StatusCode { get; set; } = String.Empty;
        public String HoldingEntityCode { get; set; } = String.Empty;
        public Boolean isCurrentStatus { get; set; }




    }
    #endregion
    #region UserInteraction
    public partial class UserInteraction
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }
        public String FromUser { get; set; } = String.Empty;
        public String ToUser { get; set; } = String.Empty;
        public String FromEntity { get; set; } = String.Empty;
        public String ToEntity { get; set; } = String.Empty;




    }
    #endregion
    #region Warehouse
    public partial class Warehouse
    {
        public String Code { get; set; } = String.Empty;
        public String WarehouseName { get; set; } = String.Empty;
        public String NameAddress { get; set; } = String.Empty;
        public String Mnemonic { get; set; } = String.Empty;
        public String? DisplayDescription { get; set; }




    }
    #endregion
    #region File Type
    public partial class FileTypeDetails
    {
        public Int32 ArchivingPeriod { get; set; }
        public Boolean IsCustomer { get; set; }
    }
    #endregion
    #region Sequence
    public partial class Sequence
    {
        public Int32 SequenceId { get; set; }
        public String Owner { get; set; } = String.Empty;
        public String? Prefix { get; set; }
        public Int64 LastIndex { get; set; }
        public String? Suffix { get; set; }
        public Boolean IsActive { get; set; }
    }
    #endregion
    #region SendEmailAttachmentDto
    public class AttachmentDto
    {
        public String FileMediaType { get; set; } = String.Empty;
        public String FileName { get; set; } = String.Empty;
        public String FileExtension { get; set; } = String.Empty;
        public Byte[] File { get; set; } = [];
    }
    #endregion
    #region SendEmailDto
    public class EmailDto
    {
        public String EmailFrom { get; set; } = String.Empty;
        public List<String> EmailTo { get; set; } = [];
        public List<String> EmailCC { get; set; } = [];
        public List<String> EmailBCC { get; set; } = [];
        public String Subject { get; set; } = String.Empty;
        public String Body { get; set; } = String.Empty;
        public Boolean IsHtml { get; set; } = false;
        public Boolean IsZipped { get; set; } = false;
        public Int32 Priority { get; set; } = 0;
        public List<AttachmentDto>? Attachments { get; set; }
    }
    #endregion

    #region ExportWareouseContainersDto
    public class ExportWareouseContainersDto
    {
        public string ContainerCode { get; set; } = string.Empty;
        public string Branch { get; set; } = String.Empty;
        public string StatusCode { get; set; } = String.Empty;
        public required string ArchivingDate { get; set; }
        public required string DestructionDate { get; set; }
        public Int32 ArchivingPeriod { get; set; }
        public string SentBy { get; set; } = String.Empty;
    }
    #endregion

    #region OldBoxDto
    public class OldBoxDto
    {
        public string Code { get; set; } = string.Empty;
        public string CompanyName { get; set; } = string.Empty;
        public int IsActive { get; set; }
        public string BoxRef { get; set; } = string.Empty;
        public string FileName { get; set; } = string.Empty;
        public string FileAdditionalInfo { get; set; } = string.Empty;
        public string BoxStatus { get; set; } = string.Empty;
        public int ArchivingPeriod { get; set; }
        public string BoxSentBy { get; set; } = string.Empty;
        public DateTime BoxSentDate { get; set; }
        public long LastIndex { get; set; }
    }
    #endregion
    #region BackfillResult
    public class BackfillResult
    {
        public int TotalContainers { get; set; }
        public int SuccessCount { get; set; }
        public int FailureCount { get; set; }
        public List<string> FailedContainers { get; set; } = new();
    }

    public class BackfillContainerInfo
    {
        public int Id { get; set; }
        public string Code { get; set; } = string.Empty;
        public string CompanyCode { get; set; } = string.Empty;
        public string Entity { get; set; } = string.Empty;
        public string StatusCode { get; set; } = string.Empty;
        public DateTime? ArchivingDate { get; set; }
        public DateTime CreatedDate { get; set; }
        public string CurrentLocation { get; set; } = string.Empty;
    }

    public class BackfillFileInfo
    {
        public int FileId { get; set; }
        public string FileName { get; set; } = string.Empty;
        public int? CustomerId { get; set; }
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public string AdditionalInfo { get; set; } = string.Empty;
        public string CompanyCode { get; set; } = string.Empty;
        public int ArchivingPeriod { get; set; }
        public string FileTypeDescription { get; set; } = string.Empty;
    }
    #endregion
}

namespace Alterna.Archive.Core.Models
{
    #region ArchiveConfiguration
    public partial class ArchiveConfiguration
    {
        public Int32 Id { get; set; }
        public String SettingName { get; set; } = String.Empty;
        public String SettingValue { get; set; } = String.Empty;
        public Boolean IsActive { get; set; }
        public String SettingDescription { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerType
    public partial class ContainerType
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String Description { get; set; } = String.Empty;
        public String Category { get; set; } = String.Empty;




    }
    #endregion
    #region Entity
    public partial class Entity
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String Description { get; set; } = String.Empty;
        public Boolean HasFullAccess { get; set; }
        public String Category { get; set; } = String.Empty;




    }
    #endregion
    #region FileName   

    public partial class FileName
    {
        public Int32 Id { set; get; }
        public String Code { get; set; } = String.Empty;
        public String Description { get; set; } = String.Empty;
        public String FileType { get; set; } = String.Empty;




    }
    #endregion
    #region FileType
    public partial class FileType
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;
        public String? Description { get; set; }
        public String Category { get; set; } = String.Empty;
        public Boolean IsBranch { get; set; }
        public Boolean HasDate { get; set; }
        public Boolean IsCustomer { get; set; }
        public Int32 ArchivingPeriod { get; set; }




    }
    #endregion
    #region Status
    public partial class Status
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String? Description { get; set; }
        public String Category { get; set; } = String.Empty;




    }
    #endregion
    #region Users
    public partial class Users
    {
        public Int32 Id { get; set; }
        public String Username { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;




    }
    #endregion
    #region Company
    public partial class Company
    {
        public String Code { get; set; } = String.Empty;
        public String CompanyName { get; set; } = String.Empty;
        public String NameAddress { get; set; } = String.Empty;
        public String Mnemonic { get; set; } = String.Empty;
        public String DisplayDescription { get; set; } = String.Empty;
        public Boolean IsBranch { get; set; }
        public Boolean IsActive { get; set; }
    }
    #endregion
    #region Container
    public partial class Container
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String CompanyCode { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;
        public String CurrentLocation { get; set; } = String.Empty;
        public String StatusCode { get; set; } = String.Empty;
        public DateTime? ArchivingDate { get; set; }
        public Boolean IsDeleted { get; set; }

        public Boolean? IsNotified { get; set; }
        public String LastModifiedBy { get; set; } = String.Empty;
        public DateTime LastModifiedDate { get; set; }
        public String CreatedBy { get; set; } = String.Empty;
        public DateTime CreatedDate { get; set; }


        // Custom Properties
        public Int32 ArchivingPeriod { get; set; }
        public DateTime? DestructionDate { get; set; }
        public List<ArchivedFile> Files { get; set; } = [];
        public String? PDF { get; set; }
        public Int32 FileCount { get; set; }
        public String SentBy { get; set; } = String.Empty;
        public String ReceivedBy { get; set; } = String.Empty;
        public DateTime? ReceivedDate { get; set; }
        public String CompanyName { get; set; } = String.Empty;
        public String ContainerType { get; set; } = String.Empty;

    }
    #endregion
    #region ContainerDisposeDocumentation
    public partial class ContainerDisposeDocumentation
    {
        public Int64 Id { get; set; }
        public Int64 ContainerDisposeRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;





    }
    #endregion
    #region ContainerDisposeRequest
    public partial class ContainerDisposeRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerPickupDocumentation
    public partial class ContainerPickupDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 ContainerPickupRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerPickupRequest
    public partial class ContainerPickupRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerRentDocumentation
    public partial class ContainerRentDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 ContainerRentRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerRentRequest
    public partial class ContainerRentRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerReturnDocumentation
    public partial class ContainerReturnDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 ContainerReturnRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region ContainerReturnRequest
    public partial class ContainerReturnRequest
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region ContainerStatus
    public partial class ContainerStatus
    {
        public Int64 Id { get; set; }
        public Int32 ContainerId { get; set; }
        public String StatusCode { get; set; } = String.Empty;
        public String HoldingEntityCode { get; set; } = String.Empty;
        public Boolean isCurrentStatus { get; set; }
        public String CreatedBy { get; set; } = String.Empty;
        public DateTime CreatedDate { get; set; }
    }
    #endregion
    #region CurrentContainerFileRelationship
    public partial class CurrentContainerFileRelationship
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }
        public Int32 ContainerId { get; set; }




    }
    #endregion
    #region Customer
    public partial class Customer
    {
        public Int32 Id { get; set; }
        public String? CompanyBook { get; set; }
        public String? ShortName { get; set; }
        public String? GivenNames { get; set; }
        public String? FamilyName { get; set; }
        public String? FatherName { get; set; }
        public String? MoFirstName { get; set; }
        public String? MoLastName { get; set; }
        public String? LegalId { get; set; }
        public String? PCntryCode { get; set; }
        public String? PhoneAreaCode { get; set; }
        public String? PhoneNo { get; set; }
        public String? MCntryCode { get; set; }
        public String? LbmbAreaCode { get; set; }
        public String? LbmbMobilenb { get; set; }
        public String? AddCustType { get; set; }





    }
    #endregion
    #region ArchivedFile
    public partial class ArchivedFile
    {
        public Int32 Id { get; set; }
        public Int32? CustomerId { get; set; }
        public String CustomerName { get; set; } = String.Empty;
        public String Name { get; set; } = String.Empty;
        public String FileTypeCode { get; set; } = String.Empty;
        public String CompanyCode { get; set; } = String.Empty;
        public String CompanyName { get; set; } = String.Empty;
        public String Status { get; set; } = String.Empty;
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? AdditionalInfo { get; set; }
        public Boolean isDeleted { get; set; }
        public DateTime CreatedDate { get; set; }
        public String CreatedBy { get; set; } = String.Empty;

        // Custom Properties
        public Int32 ContainerId { get; set; }
        public String ContainerCode { get; set; } = String.Empty;
        public List<Container> FileContainers { get; set; } = [];
        public Int32 ArchivingPeriod { get; set; } = -2;

    }
    #endregion
    #region FileClient
    public partial class FileClient
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }
        public Int32 ClientId { get; set; }




    }
    #endregion
    #region FileDisposeDocumentation
    public partial class FileDisposeDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 FileDisposeRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region FileDisposeRequest
    public partial class FileDisposeRequest
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }




    }
    #endregion
    #region FileRentDocumentation
    public partial class FileRentDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 FileRentRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region FileRentRequest
    public partial class FileRentRequest
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }




    }
    #endregion
    #region FileReturnDocumentation
    public partial class FileReturnDocumentation
    {
        public Int64 Id { get; set; }
        public Int32 FileReturnRequestId { get; set; }
        public Byte[]? Attachment { get; set; }
        public String Name { get; set; } = String.Empty;
        public Int64 Size { get; set; }
        public String Extention { get; set; } = String.Empty;
        public String MD5Hash { get; set; } = String.Empty;




    }
    #endregion
    #region FileReturnRequest
    public partial class FileReturnRequest
    {
        public Int32 Id { get; set; }
        public Int32 FileId { get; set; }




    }
    #endregion
    #region FileStatus
    public partial class FileStatus
    {
        public Int64 Id { get; set; }
        public Int32 FileId { get; set; }
        public String StatusCode { get; set; } = String.Empty;
        public String HoldingEntityCode { get; set; } = String.Empty;
        public Boolean isCurrentStatus { get; set; }




    }
    #endregion
    #region UserInteraction
    public partial class UserInteraction
    {
        public Int32 Id { get; set; }
        public Int32 ContainerId { get; set; }
        public String FromUser { get; set; } = String.Empty;
        public String ToUser { get; set; } = String.Empty;
        public String FromEntity { get; set; } = String.Empty;
        public String ToEntity { get; set; } = String.Empty;




    }
    #endregion
    #region Warehouse
    public partial class Warehouse
    {
        public String Code { get; set; } = String.Empty;
        public String WarehouseName { get; set; } = String.Empty;
        public String NameAddress { get; set; } = String.Empty;
        public String Mnemonic { get; set; } = String.Empty;
        public String? DisplayDescription { get; set; }




    }
    #endregion
    #region Sequence
    public partial class Sequence
    {
        public Int32 SequenceId { get; set; }
        public String Owner { get; set; } = String.Empty;
        public String? Prefix { get; set; }
        public Int64 LastIndex { get; set; }
        public String? Suffix { get; set; }
        public Boolean IsActive { get; set; }
    }
    #endregion

    #region AddEntityFileModel
    public partial class AddEntityFileUiModel
    {
        public Boolean CanSendContainer { get; set; }
        public Boolean IsMaxFileReached { get; set; }
    }
    #endregion

    #region OldBox
    public class OldBox
    {
        public string Code { get; set; } = string.Empty;
        public string CompanyName { get; set; } = string.Empty;
        public int IsActive { get; set; }
        public string BoxRef { get; set; } = string.Empty;
        public string FileName { get; set; } = string.Empty;
        public string FileAdditionalInfo { get; set; } = string.Empty;
        public string BoxStatus { get; set; } = string.Empty;
        public int ArchivingPeriod { get; set; }
        public string BoxSentBy { get; set; } = string.Empty;
        public string BoxSentDate { get; set; } = string.Empty;
        public long LastIndex { get; set; }
    }
    #endregion
}

using static ALTERNA.ARCHIVING.BLL.BLL;

namespace ALTERNA.ARCHIVING.BLL
{
    #region BaseRequest
    public partial class BaseRequest
    {
        public String? CorrelationId { get; set; }
        public String? CurrentUser { get; set; }
        public String? CurrentEntity { get; set; }
        public String? CurrentBranch { get; set; }
    }
    #endregion

    #region GetAll Configurations
    public partial class GetAllConfigurationsReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetAll Active Configurations
    public partial class GetActiveConfigurationsReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetCompany
    public partial class GetCompanyReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String Codes { get; set; }
    }
    #endregion

    #region GetAllCompanies
    public partial class GetAllCompaniesReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetConfiguration By Setting Name & Setting Value
    public partial class GetConfigurationReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String SettingName { get; set; }
    }
    #endregion

    #region GetContainer
    public partial class GetContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> Ids { get; set; }
    }
    #endregion

    #region GetContainerByCode
    public partial class GetContainerByCodeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<String> Codes { get; set; }
    }
    #endregion

    #region GetContainerByStatus
    public partial class GetContainerByStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String Status { get; set; }
    }
    #endregion

    #region GetBranchContainer
    public partial class GetBranchContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? StatusCode { get; set; } = String.Empty;
    }
    #endregion

    #region GetEntityContainersReq
    public partial class GetEntityContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? StatusCode { get; set; } = String.Empty;
    }
    #endregion

    #region GetContainerByEntityOrBranch
    public partial class GetContainerByEntityOrBranchReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetCustomerFilesByCustomerId
    public partial class GetCustomerFilesByCustomerIdReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public String? CustomerId { get; set; }
    }
    #endregion

    #region GetContainerFiles
    public partial class GetContainerFilesReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public Int32 ContainerId { get; set; }
    }
    #endregion

    #region GetGeneralFilesByFileType
    public partial class GetGeneralFilesByFileTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public String? FileTypeCode { get; set; }
    }
    #endregion

    #region GetContainerByFileId
    public partial class GetContainerByFileIdReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> FileIds { get; set; }
    }
    #endregion

    #region GetContainerStatus
    public partial class GetContainerStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> Ids { get; set; }
    }
    #endregion

    #region GetContainerType
    public partial class GetContainerTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetEntity
    public partial class GetEntityReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetCurrentContainerFileRelationship
    public partial class GetCurrentContainerFileRelationshipReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> Ids { get; set; }
    }
    #endregion

    #region GetCurrentFileStatusByFileId
    public partial class GetCurrentFileStatusByFileIdReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 FileId { get; set; }
    }
    #endregion

    #region Params GetCustomer By Where
    public partial class GetCustomerByWhereReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public Int32? Id { get; set; }
        public String? ShortName { get; set; }
        public String? LegalId { get; set; }
        public String? PhoneNumberString { get; set; }
    }
    #endregion

    #region GetArchivedFile
    public partial class GetArchivedFileReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> Ids { get; set; }
    }
    #endregion

    #region GetFileName
    public partial class GetFileNameReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetFilesByCustomerId
    public partial class GetFilesByCustomerIdReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 CustomerId { get; set; }
    }
    #endregion

    #region GetFileStatus
    public partial class GetFileStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> Ids { get; set; }
    }
    #endregion

    #region GetFileType
    public partial class GetFileTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetAllFileType
    public partial class GetAllFileTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetStatus
    public partial class GetStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetUserInteraction
    public partial class GetUserInteractionReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<Int32> Ids { get; set; }
    }
    #endregion

    #region GetUsers
    public partial class GetUsersReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetWarehouse
    public partial class GetWarehouseReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String Codes { get; set; }
    }
    #endregion

    #region UpdateConfiguration
    public partial class UpdateConfigurationReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String SettingName { get; set; }
        public required String SettingValue { get; set; }
        public required Boolean IsActive { get; set; }
        public required String? SettingDescription { get; set; }

    }
    #endregion

    #region DownloadPDF
    public partial class DownloadPDFReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String ContainerID { get; set; }
        public ArchivingDocumentType? DocumentType { get; set; }
    }
    #endregion

    #region DownloadDestroyedBoxPDFReq
    public partial class DownloadDestroyedBoxPDFReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String ContainerID { get; set; }
        public ArchivingDocumentType? DocumentType { get; set; }
    }
    #endregion

    #region DownloadDestroyedBoxPDF
    public partial class DownloadDestroyedBoxPDF
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String BoxReference { get; set; }


    }
    #endregion

    #region UpdateContainer
    public partial class UpdateContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String CurrentLocation { get; set; } = String.Empty;
        public String? StatusCode { get; set; } = String.Empty;
        public String? SelectedEntity { get; set; } = String.Empty;
    }
    #endregion

    #region UpdateContainerStatus
    public partial class UpdateContainerStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required Int32 ContainerId { get; set; }
        public required String StatusCode { get; set; }
        public required String HoldingEntityCode { get; set; }
        public required Boolean IsCurrentStatus { get; set; }

    }
    #endregion

    #region UpdateContainerType
    public partial class UpdateContainerTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String Code { get; set; }
        public required String Description { get; set; }
        public required String Category { get; set; }

    }
    #endregion

    #region UpdateEntity
    public partial class UpdateEntityReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String Code { get; set; }
        public String? Description { get; set; }
        public required Boolean HasFullAccess { get; set; }
        public required String Category { get; set; }

    }
    #endregion

    #region UpdateCurrentContainerFileRelationship
    public partial class UpdateCurrentContainerFileRelationshipReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required Int32 FileId { get; set; }
        public required Int32 ContainerId { get; set; }

    }
    #endregion

    #region UpdateFile
    public partial class UpdateFileReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public Int32? CustomerId { get; set; }
        public required String Name { get; set; }
        public required String FileTypeCode { get; set; }
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? AdditionalInfo { get; set; }
        public required String StatusCode { get; set; }

    }
    #endregion

    #region UpdateFileName
    public partial class UpdateFileNameReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String Code { get; set; }
        public required String Description { get; set; }
        public required String FileType { get; set; }

    }
    #endregion

    #region UpdateFileStatus
    public partial class UpdateFileStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required Int32 FileId { get; set; }
        public required String StatusCode { get; set; }
        public required String HoldingEntityCode { get; set; }
        public required Boolean isCurrentStatus { get; set; }
    }
    #endregion

    #region UpdateFileType
    public partial class UpdateFileTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String Code { get; set; }
        public required String Entity { get; set; }
        public required String Description { get; set; }
        public required String Category { get; set; }
        public required Boolean HasDate { get; set; }
        public required Boolean IsCustomer { get; set; }
        public required Int32 ArchivingPeriod { get; set; }

    }
    #endregion

    #region UpdateStatus
    public partial class UpdateStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String Code { get; set; }
        public required String Description { get; set; }
        public required String Category { get; set; }

    }
    #endregion

    #region UpdateUserInteraction
    public partial class UpdateUserInteractionReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String ContainerId { get; set; }
        public required String FromUser { get; set; }
        public required String ToUser { get; set; }
        public required String FromEntity { get; set; }
        public required String ToEntity { get; set; }

    }
    #endregion

    #region UpdateUsers
    public partial class UpdateUsersReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required String UserName { get; set; }
        public required String Entity { get; set; }

    }
    #endregion

    #region UpdateWarehouse
    public partial class UpdateWarehouseReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String Code { get; set; }
        public required String WarehouseName { get; set; }
        public required String NameAddress { get; set; }
        public required String Mnemonic { get; set; }
        public required String DisplayDescription { get; set; }

    }
    #endregion

    #region ReceiveContainer
    public partial class ReceiveContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 ContainerId { get; set; }
    }
    #endregion

    #region DeleteFile
    public partial class DeleteFileReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 FileId { get; set; }
    }
    #endregion

    #region EditContainerStatus
    public partial class EditContainerStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 ContainerId { get; set; }
        public required String StatusCode { get; set; }
        public required String HoldingEntityCode { get; set; }
        public String? Code { get; set; }
    }
    #endregion

    #region EditFileStatus
    public partial class EditFileStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 FileId { get; set; }
        public required String StatusCode { get; set; }
        public required String HoldingEntityCode { get; set; }
    }
    #endregion

    #region RemoveFileFromContainer
    public partial class RemoveFileFromContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 FileId { get; set; }
        public required Int32 ContainerId { get; set; }
    }
    #endregion

    #region ValidateCustomerReq
    public partial class ValidateCustomerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 Id { get; set; }
        public required Int32 ContainerId { get; set; }
    }
    #endregion

    #region GetWarehouseContainersReq
    public partial class GetWarehouseContainersReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? Code { get; set; }
        public String? CompanyCode { get; set; }
        public String? StatusCode { get; set; }
    }
    #endregion

    #region ExportWarehouseContainersReq
    public partial class ExportWarehouseContainersReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public List<Container> ContainerList = new List<Container>();
    }
    #endregion

    #region GetContainerArchivingPeriod
    public partial class GetContainerArchivingPeriodReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public List<Int32> ContainerIds { get; set; } = [];
    }
    #endregion

    #region GetSentContainers
    public partial class GetSentContainersReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region Get File Type Archiving Period By File Type Code
    public partial class GetFileTypeDetailsByFileTypeCodeReq
    {
        public required String FileTypeCode { get; set; }
    }
    #endregion

    #region GetContainerForEditByEntityReq
    public partial class GetContainerForEditByEntityReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetContainerForEditByRCAReq
    public partial class GetContainerForEditByRCAReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region AddFileToContainerReq
    public partial class AddFileToContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 ContainerId { get; set; }
        public Int32? CustomerId { get; set; }
        public required String CustomerName { get; set; } = String.Empty;
        public required String FileTypeCode { get; set; } = String.Empty;
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public String? AdditionalInfo { get; set; }
        public required Boolean IsEntityFile { get; set; }
        public required Boolean IsCustomerFile { get; set; }
    }
    #endregion

    #region Get Entity Container By Status Request
    public partial class GetEntityContainerByStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String Status { get; set; }
    }
    #endregion

    #region Get RCA Container By Status Request
    public partial class GetRCAContainerByStatusReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        //public required String Status { get; set; }
    }
    #endregion

    #region GetAllBranches
    public partial class GetAllBranchesReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetAllSequences
    public partial class GetAllSequencesReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region UpdateSequence
    public partial class UpdateSequenceReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 SequenceId { get; set; }
        public required String Owner { get; set; }
        public String? Prefix { get; set; }
        public required Int64 LastIndex { get; set; }
        public String? Suffix { get; set; }
        public required Boolean IsActive { get; set; }

    }
    #endregion

    #region Initialize Container Archiving Date Request
    public partial class InitializeContainerArchivingDateReq
    {
        public required Int32 Id { get; set; }
    }
    #endregion

    #region GetContainersToBeDestroyed
    public partial class ContainersToBeDestroyedReq
    {
        public required BaseRequest BaseReq { get; set; }
    }
    #endregion

    #region DestroyContainers
    public partial class DestroyContainersReq
    {
        public required BaseRequest BaseReq { get; set; }
        public required String ContainerIds { get; set; }
    }
    #endregion

    #region GetActiveCompaniesOfUser
    public partial class GetActiveCompaniesOfUserReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetEntityFileTypes
    public partial class GetEntityFileTypesReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String Code { get; set; }
    }
    #endregion

    #region GetIsEntityValidationPermittedForContainerReq
    public partial class GetIsEntityValidationPermittedForContainerReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required Int32 ContainerId { get; set; }
    }
    #endregion

    #region GetEntityFilesByFileType
    public partial class GetEntityFilesByFileTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public String? FileTypeCode { get; set; }
    }
    #endregion

    #region GetBranchFileType
    public partial class GetBranchFileTypeReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetContainerToNotifyWarehouse By RCA
    public partial class GetContainerToNotifyWarehouseReq
    {
        public required BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region GetContainerToNotifyWarehouse By Entity
    public partial class GetContainerToNotifyWarehouseByEntityReq
    {
        public required BaseRequest BaseReq { get; set; } = new BaseRequest();
    }
    #endregion

    #region SendEmailRequest
    public class SendEmailRequest
    {
        public string Username { get; set; } = String.Empty;
        public string CorrelationId { get; set; } = String.Empty;
        public string BranchName { get; set; } = String.Empty;
        public string PublicKey { get; set; } = String.Empty;
        public string ModuleName { get; set; } = String.Empty;
        public string ActivityName { get; set; } = String.Empty;
        public string ControllerName { get; set; } = String.Empty;

        public EmailDto EmailParameters { get; set; } = new();
    }
    #endregion

    #region NotifyContainersByIds
    public class NotifyContainersByIdsReq
    {
        public required BaseRequest BaseReq { get; set; } = new BaseRequest();
        public string ContainerIds { get; set; } = string.Empty;
        public required string LoggedInUserEmail { get; set; } = string.Empty;
    }
    #endregion

    #region NotifyWarehouseReq
    public class NotifyWarehouseReq
    {
        public required BaseRequest BaseReq { get; set; } = new BaseRequest();
        public string ContainerIds { get; set; } = string.Empty;
        public required string LoggedInUserEmail { get; set; } = string.Empty;
    }
    #endregion
    
    #region ExportWarehouseContainersPdfReq
    public class ExportWarehouseContainersPdfReq
    {
        public required string User { get; set; }
        public required List<ExportWareouseContainersDto> ContainerList = [];
        public string BranchList { get; set; } = "N/A";
        public required string Entity { get; set; }
    }
    #endregion

    #region ImportOldBoxesReq
    public class ImportOldBoxesReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required List<OldBoxDto> OldBoxesList = [];
    }
    #endregion

    #region BackFill missing PDF
    public partial class BackfillMissingPDFsReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        // Optional: Add filters like date range, specific containers, etc.
        public DateTime? FromDate { get; set; }
        public DateTime? ToDate { get; set; }
        public List<string>? SpecificContainers { get; set; }
    }
    #endregion
}
using System.Net;

using ALTERNA.ARCHIVING.BLL;

namespace ALTERNA.ARCHIVING.API.Controllers
{
    #region Base Response
    public partial class BaseResponse
    {
        public String CorrelationId { get; set; } = String.Empty;
        public String User { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;
        public String Branch { get; set; } = String.Empty;
        public HttpStatusCode HttpResponseCode { get; set; } = HttpStatusCode.OK;
        public String? ResponseMessage { get; set; } = String.Empty;
    }
    #endregion

    #region GetActiveConfigurations
    public partial class GetAllConfigurationsRes
    {
        public BaseResponse WebResp { get; set; } = new();
        public required GetAllConfigurationsReq Req { get; set; }
        public List<ArchiveConfiguration> Resp { get; set; } = [];
    }
    public partial class GetActiveConfigurationsRes
    {
        public BaseResponse WebResp { get; set; } = new();
        public required GetActiveConfigurationsReq Req { get; set; }
        public List<ArchiveConfiguration> Resp { get; set; } = [];
    }
    #endregion

    #region GetCompany
    public partial class GetCompanyRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetCompanyReq Req { get; set; }
        public List<Company> Resp { get; set; } = [];
    }
    #endregion

    #region GetAllCompanies
    public partial class GetAllCompaniesRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetAllCompaniesReq Req { get; set; }
        public List<Company> Resp { get; set; } = [];
    }
    #endregion

    #region GetConfiguration
    public partial class GetConfigurationRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetConfigurationReq Req { get; set; }
        public List<ArchiveConfiguration> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainer
    public partial class GetContainerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetBranchContainer
    public partial class GetBranchContainerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetBranchContainerReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region
    public partial class GetEntityContainersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetEntityContainerReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerByCode
    public partial class GetContainerByCodeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerByCodeReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerByStatus
    public partial class GetContainerByStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerByStatusReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerByEntityOrBranch
    public partial class GetContainerByEntityOrBranchRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerByEntityOrBranchReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerByFileIdRes
    public partial class GetContainerByFileIdRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerByFileIdReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerStatus
    public partial class GetContainerStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerStatusReq Req { get; set; }
        public List<ContainerStatus> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerType
    public partial class GetContainerTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerTypeReq Req { get; set; }
        public List<ContainerType> Resp { get; set; } = [];
    }
    #endregion

    #region GetEntity
    public partial class GetEntityRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetEntityReq Req { get; set; }
        public List<Entity> Resp { get; set; } = [];
    }
    #endregion

    #region GetCurrentContainerFileRelationship
    public partial class GetCurrentContainerFileRelationshipRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetCurrentContainerFileRelationshipReq Req { get; set; }
        public List<CurrentContainerFileRelationship> Resp { get; set; } = [];
    }
    #endregion

    #region GetCurrentFileStatusByFileId
    public partial class GetCurrentFileStatusByFileIdRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetCurrentFileStatusByFileIdReq Req { get; set; }
        public List<FileStatus> Resp { get; set; } = [];
    }
    #endregion

    #region GetCustomerFilesByCustomerId
    public partial class GetCustomerFilesByCustomerIdRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetCustomerFilesByCustomerIdReq Req { get; set; }
        public List<ArchivedFile> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerFilesRes
    public partial class GetContainerFilesRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerFilesReq Req { get; set; }
        public Container Resp { get; set; } = new();
    }
    #endregion

    #region GetGeneralFilesByFileType
    public partial class GetGeneralFilesByFileTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetGeneralFilesByFileTypeReq Req { get; set; }
        public List<ArchivedFile> Resp { get; set; } = [];
    }
    #endregion

    #region GetFileRes
    public partial class GetArchivedFile
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetArchivedFileReq Req { get; set; }
        public List<ArchivedFile> Resp { get; set; } = [];
    }
    #endregion

    #region Response Get Customer By Where
    public partial class GetCustomerByWhereRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetCustomerByWhereReq Req { get; set; }
        public List<Customer> Resp { get; set; } = [];
    }
    #endregion

    #region GetArchivedFile
    public partial class GetArchivedFileRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetArchivedFileReq Req { get; set; }
        public List<ArchivedFile> Resp { get; set; } = [];
    }
    #endregion

    #region GetFileName
    public partial class GetFileNameRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetFileNameReq Req { get; set; }
        public List<FileName> Resp { get; set; } = [];
    }
    #endregion

    #region GetFilesByCustomerId
    public partial class GetFilesByCustomerIdRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetFilesByCustomerIdReq Req { get; set; }
        public List<ArchivedFile> Resp { get; set; } = [];
    }
    #endregion

    #region GetFileStatus
    public partial class GetFileStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetFileStatusReq Req { get; set; }
        public List<FileStatus> Resp { get; set; } = [];
    }
    #endregion

    #region GetFileType
    public partial class GetFileTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetFileTypeReq Req { get; set; }
        public List<FileType> Resp { get; set; } = [];
    }
    #endregion

    #region GetAllFileType
    public partial class GetAllFileTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetAllFileTypeReq Req { get; set; }
        public List<FileType> Resp { get; set; } = [];
    }
    #endregion

    #region GetStatus
    public partial class GetStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetStatusReq Req { get; set; }
        public List<Status> Resp { get; set; } = [];
    }
    #endregion

    #region GetUserInteraction
    public partial class GetUserInteractionRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetUserInteractionReq Req { get; set; }
        public List<UserInteraction> Resp { get; set; } = [];
    }
    #endregion

    #region GetUsers
    public partial class GetUsersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetUsersReq Req { get; set; }
        public List<Users> Resp { get; set; } = [];
    }
    #endregion

    #region GetWarehouse
    public partial class GetWarehouseRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetWarehouseReq Req { get; set; }
        public List<Warehouse> Resp { get; set; } = [];
    }
    #endregion

    #region UpdateConfiguration
    public partial class UpdateConfigurationRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateConfigurationReq Req { get; set; }
        public ArchiveConfiguration Resp { get; set; } = new ArchiveConfiguration();
    }
    #endregion

    #region DownloadPDF
    public partial class DownloadPDFRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required DownloadPDFReq Req { get; set; }
        public String? Resp { get; set; }
    }
    #endregion

    #region DownloadPDF
    public partial class DownloadDestroyedBoxPDFRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required DownloadDestroyedBoxPDFReq Req { get; set; }
        public String? Resp { get; set; }
    }
    #endregion

    #region UpdateContainer
    public partial class UpdateContainerRes

    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateContainerReq Req { get; set; }
        public Container Resp { get; set; } = new Container();
    }
    #endregion

    #region UpdateContainerStatus
    public partial class UpdateContainerStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateContainerStatusReq Req { get; set; }
        public ContainerStatus Resp { get; set; } = new ContainerStatus();
    }
    #endregion

    #region UpdateContainerType
    public partial class UpdateContainerTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateContainerTypeReq Req { get; set; }
        public ContainerType Resp { get; set; } = new ContainerType();
    }
    #endregion

    #region UpdateEntity
    public partial class UpdateEntityRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateEntityReq Req { get; set; }
        public Entity Resp { get; set; } = new Entity();
    }
    #endregion

    #region UpdateCurrentContainerFileRelationship
    public partial class UpdateCurrentContainerFileRelationshipRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateCurrentContainerFileRelationshipReq Req { get; set; }
        public CurrentContainerFileRelationship Resp { get; set; } = new CurrentContainerFileRelationship();
    }
    #endregion

    #region UpdateFile
    public partial class UpdateFileRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateFileReq Req { get; set; }
        public ArchivedFile Resp { get; set; } = new ArchivedFile();
    }
    #endregion


    #region UpdateFileName
    public partial class UpdateFileNameRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateFileNameReq Req { get; set; }
        public FileName Resp { get; set; } = new FileName();
    }
    #endregion

    #region UpdateFileStatus
    public partial class UpdateFileStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateFileStatusReq Req { get; set; }
        public FileStatus Resp { get; set; } = new FileStatus();
    }
    #endregion

    #region UpdateFileType
    public partial class UpdateFileTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateFileTypeReq Req { get; set; }
        public FileType Resp { get; set; } = new FileType();
    }
    #endregion

    #region UpdateStatus
    public partial class UpdateStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateStatusReq Req { get; set; }
        public Status Resp { get; set; } = new Status();
    }
    #endregion

    #region UpdateUserInteraction
    public partial class UpdateUserInteractionRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateUserInteractionReq Req { get; set; }
        public UserInteraction Resp { get; set; } = new UserInteraction();
    }
    #endregion

    #region UpdateUsers
    public partial class UpdateUsersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateUsersReq Req { get; set; }
        public Users Resp { get; set; } = new Users();
    }
    #endregion

    #region UpdateWarehouse
    public partial class UpdateWarehouseRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateWarehouseReq Req { get; set; }
        public Warehouse Resp { get; set; } = new Warehouse();
    }
    #endregion

    #region ReceiveContainer
    public partial class ReceiveContainerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required ReceiveContainerReq Req { get; set; }
        public Container Resp { get; set; } = new Container();
    }
    #endregion

    #region DeleteFile
    public partial class DeleteFileRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required DeleteFileReq Req { get; set; }
        public Boolean Resp { get; set; }
    }
    #endregion

    #region EditContainerStatus
    public partial class EditContainerStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required EditContainerStatusReq Req { get; set; }
        public Container? Resp { get; set; }
    }
    #endregion

    #region EditFileStatus
    public partial class EditFileStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required EditFileStatusReq Req { get; set; }
        public ArchivedFile? Resp { get; set; }
    }
    #endregion

    #region RemoveFileFromContainer
    public partial class RemoveFileFromContainerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required RemoveFileFromContainerReq Req { get; set; }
        public Boolean Resp { get; set; }
    }
    #endregion

    #region ValidateCustomer
    public partial class ValidateCustomerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required ValidateCustomerReq Req { get; set; }
        public String? Resp { get; set; }
    }
    #endregion

    #region GetWarehouseContainers
    public partial class GetWarehouseContainersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetWarehouseContainersReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerArchivingPeriod
    public partial class GetContainerArchivingPeriodRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerArchivingPeriodReq Req { get; set; }
        public List<ContainerArchivingPeriod> Resp { get; set; } = [];
    }
    #endregion

    #region GetSentContainers
    public partial class GetSentContainersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetSentContainersReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region Get Entity Container By Status
    public partial class GetEntityContainerByStatusResp
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetEntityContainerByStatusReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region Get RCA Container By Status
    public partial class GetRCAContainerByStatusResp
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetRCAContainerByStatusReq Req { get; set; }
        public List<Container>? Resp { get; set; }
    }
    #endregion

    #region GetContainerForEditByEntityRes
    public partial class GetContainerForEditByEntityRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerForEditByEntityReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerForEditByRCARes
    public partial class GetContainerForEditByRCARes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerForEditByRCAReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region AddFileToContainerRes
    public partial class AddFileToContainerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required AddFileToContainerReq Req { get; set; }
        public Container Resp { get; set; } = new();
    }
    #endregion

    #region GetAllBranches
    public partial class GetAllBranchesRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetAllBranchesReq Req { get; set; }
        public List<Company> Resp { get; set; } = [];
    }
    #endregion

    #region GetAllSequences
    public partial class GetAllSequencesRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetAllSequencesReq Req { get; set; }
        public List<Sequence> Resp { get; set; } = [];
    }
    #endregion

    #region UpdateSequence
    public partial class UpdateSequenceRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required UpdateSequenceReq Req { get; set; }
        public Sequence Resp { get; set; } = new Sequence();
    }
    #endregion

    #region GetContainersToBeDestroyed
    public partial class GetContainersToBeDestroyedRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required ContainersToBeDestroyedReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region DestroyContainers
    public partial class DestroyContainersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required DestroyContainersReq Req { get; set; }
        public List<Container> Resp { get; set; } = [];
    }
    #endregion

    #region GetActiveCompaniesOfUser
    public partial class GetActiveCompaniesOfUserRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetActiveCompaniesOfUserReq Req { get; set; }
        public List<Company> Resp { get; set; } = [];
    }
    #endregion

    #region GetEntityFileTypes
    public partial class GetEntityFileTypesRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetEntityFileTypesReq Req { get; set; }
        public List<FileType> Resp { get; set; } = [];
    }
    #endregion

    #region GetIsEntityValidationPermittedForContainer
    public partial class GetIsEntityValidationPermittedForContainerRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetIsEntityValidationPermittedForContainerReq Req { get; set; }
        public Boolean Resp { get; set; }
    }
    #endregion

    #region GetEntityFilesByFileType
    public partial class GetEntityFilesByFileTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetEntityFilesByFileTypeReq Req { get; set; }
        public List<ArchivedFile> Resp { get; set; } = [];
    }
    #endregion

    #region GetBranchFileType
    public partial class GetBranchFileTypeRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetBranchFileTypeReq Req { get; set; }
        public List<FileType> Resp { get; set; } = [];
    }
    #endregion

    #region GetContainerToNotifyWarehouseRes ByRCA
    public partial class GetContainerToNotifyWarehouseRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerToNotifyWarehouseReq Req { get; set; }
        public List<Container>? Resp { get; set; }
    }
    #endregion

    #region GetContainerToNotifyWarehouseRes By Entity
    public partial class GetContainerToNotifyWarehouseByEntityRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required GetContainerToNotifyWarehouseByEntityReq Req { get; set; }
        public List<Container>? Resp { get; set; }
    }
    #endregion

    #region NotifyContainersByIds
    public partial class NotifyContainersByIdsRes
    {
        public BaseResponse WebResp { get; set; } = new();
        public required NotifyContainersByIdsReq Req { get; set; }
    }
    #endregion

    #region ExportWarehouseContainersRes
    public partial class ExportWarehouseContainersRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public ExportWarehouseContainersReq? Req { get; set; }
        public String Resp { get; set; } = String.Empty;
    }
    #endregion

    #region ImportOldBoxesRes
    public partial class ImportOldBoxesRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public ImportOldBoxesReq? Req { get; set; }
    }
    #endregion

    #region BackfillMissingPDFsRes
    public partial class BackfillMissingPDFsRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required BackfillMissingPDFsReq Req { get; set; }
        public ALTERNA.ARCHIVING.BLL.BackfillResult? Resp { get; set; }
    }
    #endregion
}


