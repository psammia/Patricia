        #region Insert User Application With Files
        public async Task<Application> Insert_UserApplication_WithFiles(Insert_UserApplication_WithFiles_Request request)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();
            parameters.Add("P__CorrelationId", request.CorrelationId);
            parameters.Add("P__External_Id", request.External_Id);
            parameters.Add("P__TVP_Files", GetAppFilesDt(request.app_FilesList).AsTableValuedParameter());


            IEnumerable<Application> res = await dal.ExecuteQueryAsync<Application>(
                "usp_InsertUserApplicationWithFiles",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            return res.ToList().First();

        }

        private DataTable GetAppFilesDt(List<App_Files> app_Files)
        {
            DataTable dt = new DataTable("TVP_Files");
            dt.Columns.Add("File_Name");
            dt.Columns.Add("File_Type");
            dt.Columns.Add("File_Size");
            dt.Columns.Add("File_Data");

            foreach (App_Files appFile in app_Files)
            {
                DataRow dr = dt.NewRow();

                dr["File_Name"] = appFile.File_Name;
                dr["File_Type"] = appFile.File_Type;
                dr["File_Size"] = appFile.File_Size;
                dr["File_Data"] = appFile.File_Data;

                dt.Rows.Add(dr);
            }
            return dt;
        }
