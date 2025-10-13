        public String DownloadPDF(DownloadPDFReq downloadPDFReq)
        {
            OnPreEventDownloadPDF?.Invoke(ref downloadPDFReq);

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
