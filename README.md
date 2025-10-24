        [HttpPost]
public ActionResult ImportOldBoxesArchive(IFormFile excelFile)
{
    string correlationId = Guid.NewGuid().ToString();

    try
    {
        var validationConfig = new ExcelValidationConfig
        {
            // ... your config
        };

        // Validate the file
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
