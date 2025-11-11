3-+-----------------------
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
2--------------------------
As for the step 2 i have this version of Download API in the backend in Archiving Project:
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

 Question: Give the right version using the same pattern to resolve the previous issue when trying to download pdf

 Step 3 : Provide me with exact code.



1---------------------
An unhandled exception occurred while processing the request.
ErrorHandler: Exception of type 'Alterna.Archive.Core.Global.ErrorHandler' was thrown.
Alterna.Archive.Core.Controllers.FilesController.ReDownloadSendPDF(string boxReference) in FilesController.cs, line 209
Stack Query Cookies Headers Routing
ErrorHandler: Exception of type 'Alterna.Archive.Core.Global.ErrorHandler' was thrown.
Alterna.Archive.Core.Controllers.FilesController.ReDownloadSendPDF(string boxReference) in FilesController.cs
+
                    throw new ErrorHandler(new ErrorModel()
lambda_method157(Closure , object , object[] )
Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor+SyncActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, object controller, object[] arguments)
Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Logged|12_1(ControllerActionInvoker invoker)
Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, object state, bool isCompleted)
Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(ref State next, ref Scope scope, ref object state, ref bool isCompleted)
Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeInnerFilterAsync()
Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeNextResourceFilter>g__Awaited|25_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, object state, bool isCompleted)
Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.Rethrow(ResourceExecutedContextSealed context)
Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.Next(ref State next, ref Scope scope, ref object state, ref bool isCompleted)
Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.InvokeFilterPipelineAsync()
Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
Microsoft.AspNetCore.Diagnostics.StatusCodePagesMiddleware.Invoke(HttpContext context)
Microsoft.AspNetCore.Session.SessionMiddleware.Invoke(HttpContext context)
Microsoft.AspNetCore.Session.SessionMiddleware.Invoke(HttpContext context)
Microsoft.AspNetCore.Authorization.AuthorizationMiddleware.Invoke(HttpContext context)
Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddlewareImpl.Invoke(HttpContext context)Show moreThe error message is not very descriptive. We need to see what the actual exception is. Let me help you add better error logging and handling to identify the root cause.
Step 1: Add Detailed Logging to FileController
Update your ReDownloadSendPDF method in FilesController.cs to log the actual error:
csharp
public ActionResult ReDownloadSendPDF(String boxReference)
{
    string correlationId = Guid.NewGuid().ToString();
    
    try
    {
        DownloadPDFModel model = new();
        DownloadPDFRes downloadPDFRes = Common.ApiCall<DownloadPDFRes>(new DownloadPDFReq()
        {
            BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
            ContainerID = boxReference
        }, "DownloadPDF");

        // Log the response for debugging
        System.Diagnostics.Debug.WriteLine($"PDF Response - IsNull: {downloadPDFRes.Resp is null}, Length: {downloadPDFRes.Resp?.Length ?? 0}");
        System.Diagnostics.Debug.WriteLine($"HTTP Response Code: {downloadPDFRes.WebResp.HttpResponseCode}");
        System.Diagnostics.Debug.WriteLine($"Response Message: {downloadPDFRes.WebResp.ResponseMessage}");

        if (downloadPDFRes.Resp is null || downloadPDFRes.Resp.Length == 0)
        {
            string errorMsg = $"Invalid PDF or PDF not found for container: {boxReference}. Response Message: {downloadPDFRes.WebResp.ResponseMessage}";
            
            HttpContext.Session.SetString("CorrelationId", downloadPDFRes.WebResp.CorrelationId);
            HttpContext.Session.SetString("ErrorMessage", errorMsg);

            // Log the actual error
            System.Diagnostics.Debug.WriteLine($"ERROR: {errorMsg}");

            throw new ErrorHandler(new ErrorModel() 
            { 
                ErrorCorrelationId = downloadPDFRes.WebResp.CorrelationId, 
                ErrorMessage = errorMsg
            });
        }

        String PDF = downloadPDFRes.Resp ?? String.Empty;

        if (String.IsNullOrWhiteSpace(PDF))
        {
            string errorMsg = $"Empty PDF response for container: {boxReference}";
            
            HttpContext.Session.SetString("CorrelationId", downloadPDFRes.WebResp.CorrelationId);
            HttpContext.Session.SetString("ErrorMessage", errorMsg);

            System.Diagnostics.Debug.WriteLine($"ERROR: {errorMsg}");

            throw new ErrorHandler(new ErrorModel() 
            { 
                ErrorCorrelationId = downloadPDFRes.WebResp.CorrelationId, 
                ErrorMessage = "PDF Server returned empty response" 
            });
        }

        // Validate that PDF string contains valid hexadecimal characters
        if (!System.Text.RegularExpressions.Regex.IsMatch(PDF, @"^[0-9A-Fa-f]+$"))
        {
            string errorMsg = $"Invalid PDF format for container: {boxReference}. PDF does not contain valid hexadecimal characters.";
            
            HttpContext.Session.SetString("CorrelationId", downloadPDFRes.WebResp.CorrelationId);
            HttpContext.Session.SetString("ErrorMessage", errorMsg);

            System.Diagnostics.Debug.WriteLine($"ERROR: {errorMsg}");
            System.Diagnostics.Debug.WriteLine($"PDF Content (first 100 chars): {PDF.Substring(0, Math.Min(100, PDF.Length))}");

            throw new ErrorHandler(new ErrorModel() 
            { 
                ErrorCorrelationId = downloadPDFRes.WebResp.CorrelationId, 
                ErrorMessage = "PDF response is not in valid hexadecimal format" 
            });
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
    catch (ErrorHandler)
    {
        throw; // Re-throw ErrorHandler exceptions as-is
    }
    catch (Exception ex)
    {
        // Log the full exception details
        System.Diagnostics.Debug.WriteLine($"CRITICAL ERROR in ReDownloadSendPDF for {boxReference}:");
        System.Diagnostics.Debug.WriteLine($"Exception Type: {ex.GetType().Name}");
        System.Diagnostics.Debug.WriteLine($"Exception Message: {ex.Message}");
        System.Diagnostics.Debug.WriteLine($"Stack Trace: {ex.StackTrace}");
        if (ex.InnerException != null)
        {
            System.Diagnostics.Debug.WriteLine($"Inner Exception: {ex.InnerException.Message}");
        }

        HttpContext.Session.SetString("CorrelationId", correlationId);
        HttpContext.Session.SetString("ErrorMessage", ex.Message);

        throw new ErrorHandler(new ErrorModel() 
        { 
            ErrorCorrelationId = correlationId, 
            ErrorMessage = $"Failed to download PDF: {ex.Message}" 
        });
    }
}
Step 2: Check the Backend API - DownloadPDF Endpoint
The issue might be in the Archiving Project backend. Make sure your DownloadPDF endpoint is properly calling the PDF Generator's RedownloadDocPDFForArchive method.
Can you share your DownloadPDF method from the Archiving project backend (Archiving.cs or similar)?
It should look something like this:
csharp
[HttpPost]
[Route("DownloadPDF")]
public DownloadPDFRes DownloadPDF(DownloadPDFReq req)
{
    DownloadPDFRes response = new DownloadPDFRes()
    {
        Req = req,
    };

    CorrelationInfo correlationInfo = new()
    {
        CorrelationId = req.BaseReq.CorrelationId,
        RDirection = RequestDirection.Request,
        RequestURL = "DownloadPDF",
        UserName = req.BaseReq.CurrentUser
    };

    try
    {
        LogInfo("DownloadPDF Has been called", correlationInfo);

        // Call PDF Generator Service
        RedownloadDocPDFForArchiveRequest pdfRequest = new()
        {
            ContainerID = req.ContainerID
        };

        String data = JsonConvert.SerializeObject(pdfRequest);
        HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
        HttpClient client = new();
        String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ?? 
            throw new Exception("PDF Service not initialized");

        Task<HttpResponseMessage> Request = client.PostAsync(
            $"{PDFRequestBase}RedownloadDocPDFForArchive", content);

        Request.Wait();
        Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
        responseString.Wait();

        response.Resp = responseString.Result;
        response.WebResp.CorrelationId = req.BaseReq.CorrelationId;
        response.WebResp.User = req.BaseReq.CurrentUser;
        response.WebResp.HttpResponseCode = HttpStatusCode.OK;

        return response;
    }
    catch (Exception ex)
    {
        response.WebResp.CorrelationId = req.BaseReq.CorrelationId;
        response.WebResp.User = req.BaseReq.CurrentUser;
        response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
        response.WebResp.ResponseMessage = ex.Message;

        LogError(ex.Message, correlationInfo, ex);

        return response;
    }
}
Step 3: Verify PDF Generator Controller
Make sure your PDF Generator BaseController has the RedownloadDocPDFForArchive endpoint properly set up as you showed earlier.
Step 4: Check Your App Logs
After adding the debug logging, run the application and check:

Visual Studio Output window (Debug messages)
Application logs
IIS logs (if using IIS)

The debug statements will tell us exactly where the error is occurring and what the actual error message is.
Can you run the application with these logging changes and share:

The debug output from Visual Studio's Output window
The actual error message that gets logged
Your DownloadPDF backend method if it's different from what I showed above

This will help us pinpoint the exact issue!
