== Request.cs
public class EntityDocRequest
{
    public required string DestructionDate { get; set; }
    public required string User { get; set; }
    public string BranchList { get; set; } = "N/A";
    public required string Entity { get; set; }
    public required string ContainerID { get; set; }
    public List<EntityFile> EntityFiles { get; set; } = [];
    public required string CreationDate { get; set; }
}
=== Model.cs
public class EntityFile
{
    public required string DocumentType { get; set; }
    public string DocumentDescription { get; set; } = string.Empty;
}

=== BLL.cs
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

        private byte[] GetByteArrayForEntityDocPDFForArchive(EntityDocRequest entityDocRequest)
    {
        Settings.License = LicenseType.Community;
        var FontsFamily = ConfigurationManager.AppSettings["FONT_FAMILY"] ?? "Times New Roman";
        if (!float.TryParse(ConfigurationManager.AppSettings["FONT_SIZE"], out var FontSize)) FontSize = 14f;
        string[] FontFamilyList = FontsFamily.Split(',');

        var retRes = Document.Create(container =>
        {
            container.Page(page =>
            {
                page.Size(PageSizes.A4);
                page.Margin(15);
                page.DefaultTextStyle(x =>
                    x.FontFamily(FontFamilyList).FontSize(FontSize));
                page.Header().Element(h =>
                {
                    h.Table(t =>
                    {
                        t.ColumnsDefinition(col =>
                        {
                            col.RelativeColumn();
                            col.RelativeColumn();
                        });
                        t.Header(th =>
                        {
                            th.Cell().ColumnSpan(2).Element(HeadMid).Text("SUMMARY OF DELIVERY TO ARCHIVES")
                                .SemiBold().FontSize(FontSize + 2);
                        });

                        t.Cell().Column(1).Row(2).Element(HeadLStart).Text($"Date: {entityDocRequest.CreationDate}");
                        t.Cell().Column(1).Row(3).Element(HeadL)
                            .Text($"Destruction Date: {entityDocRequest.DestructionDate}");
                        t.Cell().Column(1).Row(4).Element(HeadL).Text($"User: {entityDocRequest.User}");
                        t.Cell().Column(1).Row(5).Element(HeadLEnd).Text($"Entity: {entityDocRequest.Entity}");
                        t.Cell().Column(2).Row(2).RowSpan(4).Element(HeadSpan).Text($"{entityDocRequest.ContainerID}")
                            .FontSize(FontSize * 3f).FontColor(Color.FromARGB(180, 0, 0, 0)).Bold();
                    });
                });
                page.Content()
                    .Column(x =>
                    {
                        x.Item().Table(table =>
                        {
                            table.ColumnsDefinition(columns =>
                            {
                                columns.RelativeColumn();
                                columns.RelativeColumn();
                            });
                            table.Header(header =>
                                {
                                    header.Cell().Row(1).Column(1).Element(HeaderC).Text("Document type")
                                        .FontSize(FontSize + 2);
                                    header.Cell().Row(1).Column(2).Element(HeaderC).Text("Document Description")
                                        .FontSize(FontSize + 2);
                                }
                            );

                            uint i = 1;
                            foreach (var entityFile in entityDocRequest.EntityFiles)
                            {
                                table.Cell().Row(i).Column(1).Element(DocumentType).Text(entityFile.DocumentType);
                                table.Cell().Row(i).Column(2).Element(BlockEntity).Text(entityFile.DocumentDescription);
                                i++;
                            }

                            table.Footer(footer =>
                                {
                                    footer.Cell().ColumnSpan(2).Element(FooterR)
                                        .Text("Branch / Entity signature and seal");
                                }
                            );
                        });
                    });
                page.Footer()
                    .AlignCenter()
                    .Text(x =>
                    {
                        x.Span("Page ");
                        x.CurrentPageNumber();
                        x.Span(" Of ");
                        x.TotalPages();
                    });
            });
        }).GeneratePdf();

        return retRes;
    }

    == Sql Procedure
      ALTER PROCEDURE [dbo].[usp_InsertPDF] (
    @PDF VARBINARY(MAX),
    @Request NVARCHAR(MAX),
    @ApiMethod NVARCHAR(500),
    @BranchList NVARCHAR(MAX),
    @Entity NVARCHAR(10),
    @User NVARCHAR(250)
  ) AS BEGIN
SET
  NOCOUNT ON;

INSERT INTO
  t_PDF (
    PDF,
    Request,
    ApiMethod,
    BranchList,
    Entity,
    CreatedBy,
    LastModifiedBy
  )
VALUES
  (
    @PDF,
    @Request,
    @ApiMethod,
    @BranchList,
    @Entity,
    @User,
    @User
  )
END

=== 
    
