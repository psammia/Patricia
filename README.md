=== Front End Controller = ConfigurationController
       
       [HttpPost]
       public ActionResult ImportOldBoxesArchive(IFormFile excelFile)
       {
           string correlationId = Guid.NewGuid().ToString();

           try
           {
               if (!ValidateFiles(excelFile, out string message))
               {
                   return Json(new { isSuccess = false, message = message, correlationId = correlationId });
               }

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

       private bool ValidateFiles(IFormFile? excelFile, out string message)
       {
           message = string.Empty;

           if (excelFile == null || excelFile.Length == 0)
           {
               message = "Excel file is required.";

               return false;
           }

           string generalExtension = Path.GetExtension(excelFile.FileName).ToLower();

           if (generalExtension != ".xlsx")
           {
               message = "file must have a .xlsx extension.";

               return false;
           }

           return true;
       }

       


