        #region Generate Warehouse Containers Report
        public byte[] GenerateWarehouseContainersReport(ExportWarehouseContainersViewModel vm)
        {
            String data = JsonConvert.SerializeObject(vm);
            HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
            HttpClient client = new();
            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ??
                                    throw new SGBLInternalServerException(
                                        "PDF Service not initialized please Contact Support");

            Task<HttpResponseMessage>
                Request = client.PostAsync($"{PDFRequestBase}ExportWarehouseContainers", content);

            Request.Wait();
            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
            responseString.Wait();

            string result = responseString.Result;

            Byte[] bytearray = new Byte[result.Length / 2];
            for (Int32 i = 0; i < result.Length; i += 2)
            {
                bytearray[i / 2] = Convert.ToByte(result.Substring(i, 2), 16);
            }
            return bytearray;
        }
        #endregion
