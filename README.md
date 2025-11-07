    public byte[] GenerateEntityDocPDFForArchive(EntityDocRequest entityDocRequest)
    {
        var retRes = GetByteArrayForEntityDocPDFForArchive(entityDocRequest);

        byte[] empty = [];

        DynamicParameters dynamicParameters = new();
        dynamicParameters.Add("PDF", empty, DbType.Binary, ParameterDirection.Input);
        dynamicParameters.Add("Request", JsonConvert.SerializeObject(entityDocRequest, Formatting.Indented),
            DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("ApiMethod", "GenerateBranchDocPDFForArchive", DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("BranchList", entityDocRequest.BranchList, DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("Entity", entityDocRequest.Entity, DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("User", entityDocRequest.User, DbType.String, ParameterDirection.Input);

        using (DAL.DAL dal = new(Catalog_Archive, out var res))
        {
            var command = ConfigurationManager.AppSettings["Insert_PDF_SP"] ?? "usp_InsertPDF";
            dal.ExecuteQuery(command, dynamicParameters);
        }

        return retRes;
    }
	
