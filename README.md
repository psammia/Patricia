Using .net Core C# 
I have two project, from the first i extract data, the second is to generate PDF using QuestPDF.
Using SQL as Database

*) Here's the schema for tables
USE [Alterna.Archive]
GO
/****** Object:  Table [dbo].[Configuration]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[Configuration](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[SettingName] [nvarchar](50) NOT NULL,
	[SettingValue] [nvarchar](50) NOT NULL,
	[IsActive] [bit] NOT NULL,
	[SettingDescription] [nvarchar](1000) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_Configuration] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[lkp_Entity]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[lkp_Entity](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](10) NOT NULL,
	[Description] [nvarchar](250) NULL,
	[HasFullAccess] [bit] NOT NULL,
	[Category] [nvarchar](100) NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_lkp_Entity] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[lkp_FileType]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[lkp_FileType](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](10) NOT NULL,
	[Entity] [nvarchar](10) NOT NULL,
	[Category] [nvarchar](10) NOT NULL,
	[Description] [nvarchar](250) NOT NULL,
	[HasDate] [bit] NOT NULL,
	[IsCustomer] [bit] NOT NULL,
	[ArchivingPeriod] [int] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_lkp_FileType] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[lkp_Status]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[lkp_Status](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](10) NOT NULL,
	[Description] [nvarchar](100) NULL,
	[Category] [nvarchar](10) NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_lkp_Status] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Company]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Company](
	[Code] [nvarchar](11) NOT NULL,
	[CompanyName] [nvarchar](22) NOT NULL,
	[NameAddress] [nvarchar](35) NOT NULL,
	[Mnemonic] [nvarchar](50) NOT NULL,
	[DisplayDescription] [nvarchar](250) NULL,
	[isBranch] [bit] NOT NULL,
	[IsActive] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Company] PRIMARY KEY CLUSTERED 
(
	[Code] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Container]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Container](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](50) NOT NULL,
	[CompanyCode] [nvarchar](9) NOT NULL,
	[Entity] [nvarchar](10) NOT NULL,
	[CurrentLocation] [nvarchar](50) NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[ArchivingDate] [datetime] NULL,
	[isDeleted] [bit] NOT NULL,
	[isNotified] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Container] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_ContainerStatus]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_ContainerStatus](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[ContainerId] [int] NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[HoldingEntityCode] [nvarchar](11) NOT NULL,
	[isCurrentStatus] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_ContainerStatus] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_CurrentContainerFileRelationship]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_CurrentContainerFileRelationship](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[FileId] [int] NOT NULL,
	[ContainerId] [int] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_CurrentContainerFileRelationship] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Customer]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Customer](
	[Id] [int] NOT NULL,
	[CompanyBook] [nvarchar](11) NULL,
	[ShortName] [nvarchar](max) NULL,
	[GivenNames] [nvarchar](max) NULL,
	[FamilyName] [nvarchar](max) NULL,
	[FatherName] [nvarchar](max) NULL,
	[MoFirstName] [nvarchar](max) NULL,
	[MoLastName] [nvarchar](max) NULL,
	[LegalId] [nvarchar](max) NULL,
	[PCntryCode] [nvarchar](max) NULL,
	[PhoneAreaCode] [nvarchar](max) NULL,
	[PhoneNo] [nvarchar](max) NULL,
	[MCntryCode] [nvarchar](max) NULL,
	[LbmbAreaCode] [nvarchar](max) NULL,
	[LbmbMobilenb] [nvarchar](max) NULL,
	[AddCustType] [nvarchar](max) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Client] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_File]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_File](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[CustomerId] [int] NULL,
	[Name] [nvarchar](250) NOT NULL,
	[FileTypeCode] [nvarchar](10) NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[CompanyCode] [nvarchar](9) NOT NULL,
	[FromDate] [date] NULL,
	[ToDate] [date] NULL,
	[AdditionalInfo] [nvarchar](1000) NULL,
	[isDeleted] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_File] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_FileStatus]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_FileStatus](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[FileId] [int] NOT NULL,
	[StatusCode] [nvarchar](10) NOT NULL,
	[HoldingEntityCode] [nvarchar](11) NOT NULL,
	[isCurrentStatus] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_FileStatus] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_PDF]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_PDF](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[PDF] [varbinary](max) NOT NULL,
	[Request] [nvarchar](max) NOT NULL,
	[ApiMethod] [nvarchar](500) NOT NULL,
	[BranchList] [nvarchar](max) NULL,
	[Entity] [nvarchar](10) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_PDF] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Sequence]    Script Date: 17/10/2025 11:05:56 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Sequence](
	[SequenceId] [int] IDENTITY(1,1) NOT NULL,
	[Owner] [nvarchar](50) NOT NULL,
	[Prefix] [nvarchar](50) NULL,
	[LastIndex] [bigint] NOT NULL,
	[Suffix] [nvarchar](50) NULL,
	[IsActive] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Sequence] PRIMARY KEY CLUSTERED 
(
	[SequenceId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY],
 CONSTRAINT [IX_t_Sequence] UNIQUE NONCLUSTERED 
(
	[Owner] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
ALTER TABLE [dbo].[Configuration] ADD  CONSTRAINT [DF__Configura__Creat__55009F39]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[Configuration] ADD  CONSTRAINT [DF__Configura__LastM__55F4C372]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[lkp_Entity] ADD  CONSTRAINT [DF_lkp_Entity_HasFullAccess]  DEFAULT ((0)) FOR [HasFullAccess]
GO
ALTER TABLE [dbo].[lkp_Entity] ADD  CONSTRAINT [DF_lkp_Entity_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[lkp_Entity] ADD  CONSTRAINT [DF_lkp_Entity_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[lkp_FileType] ADD  CONSTRAINT [DF__lkp_FileT__Creat__160F4887]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[lkp_FileType] ADD  CONSTRAINT [DF__lkp_FileT__LastM__17036CC0]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[lkp_Status] ADD  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[lkp_Status] ADD  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Company] ADD  CONSTRAINT [DF__t_Company__isBra__4D5F7D71]  DEFAULT ((1)) FOR [isBranch]
GO
ALTER TABLE [dbo].[t_Company] ADD  CONSTRAINT [DF__t_Company__IsAct__7A3223E8]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[t_Company] ADD  CONSTRAINT [DF_t_Company_CreatedBy]  DEFAULT ('ETLSysUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Company] ADD  CONSTRAINT [DF__t_Company__Creat__19DFD96B]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Company] ADD  CONSTRAINT [DF_t_Company_LastModifiedBy]  DEFAULT ('ETLSysUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Company] ADD  CONSTRAINT [DF__t_Company__LastM__1AD3FDA4]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF_t_Container_isDeleted]  DEFAULT ((0)) FOR [isDeleted]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF_t_Container_isNotified]  DEFAULT ((0)) FOR [isNotified]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF__t_Contain__Creat__1BC821DD]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Container] ADD  CONSTRAINT [DF__t_Contain__LastM__1CBC4616]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_ContainerStatus] ADD  CONSTRAINT [DF_t_ContainerStatus_isCurrentStatus]  DEFAULT ((1)) FOR [isCurrentStatus]
GO
ALTER TABLE [dbo].[t_ContainerStatus] ADD  CONSTRAINT [DF__t_Contain__Creat__25518C17]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_ContainerStatus] ADD  CONSTRAINT [DF__t_Contain__LastM__2645B050]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_CurrentContainerFileRelationship] ADD  CONSTRAINT [DF__t_Current__Creat__2739D489]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_CurrentContainerFileRelationship] ADD  CONSTRAINT [DF__t_Current__LastM__282DF8C2]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF_t_Customer_CreatedBy]  DEFAULT ('ETLSysUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF__t_Custome__Creat__29221CFB]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF_t_Customer_LastModifiedBy]  DEFAULT ('ETLSysUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Customer] ADD  CONSTRAINT [DF__t_Custome__LastM__2A164134]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF_t_File_FromDate]  DEFAULT (getdate()) FOR [FromDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF_t_File_ToDate]  DEFAULT (getdate()) FOR [ToDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF_t_File_isDeleted]  DEFAULT ((0)) FOR [isDeleted]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF__t_File__CreatedD__2B0A656D]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_File] ADD  CONSTRAINT [DF__t_File__LastModi__2BFE89A6]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_FileStatus] ADD  CONSTRAINT [DF_t_FileStatus_isCurrentStatus]  DEFAULT ((1)) FOR [isCurrentStatus]
GO
ALTER TABLE [dbo].[t_FileStatus] ADD  CONSTRAINT [DF__t_FileSta__Creat__32AB8735]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_FileStatus] ADD  CONSTRAINT [DF__t_FileSta__LastM__339FAB6E]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_PDF] ADD  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_PDF] ADD  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Sequence] ADD  CONSTRAINT [DF_t_Sequence_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[t_Sequence] ADD  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Sequence] ADD  DEFAULT (getdate()) FOR [LastModifiedDate]
GO

1) Project No.1 Name: Archiving that communicates with project No.1 through API call
2) Project No.2 Name: PdfGenerator  
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
public class RedownloadDocPDFForArchiveRequest
{
    public required string ContainerID { get; set; }
    public required ArchivingDocumentType DocumentType { get; set; }
}

=== BLL.cs
using System.Configuration;
using System.Data;
using System.Net;

using Dapper;

using Newtonsoft.Json;

using QuestPDF;
using QuestPDF.Fluent;
using QuestPDF.Helpers;
using QuestPDF.Infrastructure;

    public byte[] RedownloadDocPDFForArchive(RedownloadDocPDFForArchiveRequest redownloadDocPDFForArchiveRequest)
    {
        byte[] retRes = [];
        byte[] pdfInDb = [];

        DynamicParameters dynamicParameters = new();
        dynamicParameters.Add("BoxReference", redownloadDocPDFForArchiveRequest.ContainerID);

        var originalRequestInJsonFormat = string.Empty;

        using (DAL.DAL dal = new(Catalog_Archive, out var res))
        {
            var command = "";
            switch (redownloadDocPDFForArchiveRequest.DocumentType)
            {
                case ArchivingDocumentType.ENTITY_PDF:
                    command = ConfigurationManager.AppSettings["Get_PDF_Var_Binary_By_Box_Reference_SP"] ??
                              "usp_GetPDFVarBinaryByBoxReference";
                    pdfInDb = dal.ExecuteQuery<byte[]>(command, dynamicParameters).DefaultIfEmpty([]).First();

                    if (pdfInDb.Length > 5)
                    {
                        retRes = pdfInDb;
                    }
                    else
                    {
                        command = ConfigurationManager.AppSettings["Get_PDF_Request_By_Box_Reference_SP"] ??
                                  "usp_GetPDFRequestByBoxReference";
                        originalRequestInJsonFormat = dal.ExecuteQuery<string>(command, dynamicParameters)
                            .DefaultIfEmpty(string.Empty).First();

                        retRes = GetByteArrayForEntityDocPDFForArchive(
                            JsonConvert.DeserializeObject<EntityDocRequest>(originalRequestInJsonFormat)!);
                    }

                    break;
            }
        }

        return retRes;
    }
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

USE [Alterna.Archive]
GO
/****** Object:  StoredProcedure [dbo].[usp_GetPDFVarBinaryByBoxReference]    Script Date: 16/10/2025 2:29:02 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[usp_GetPDFVarBinaryByBoxReference]
	@BoxReference nvarchar(max)
AS
BEGIN

Select PDF from t_PDF where Request like '% "ContainerID": "'+@BoxReference+'"%' AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive');

END

USE [Alterna.Archive]
GO
/****** Object:  StoredProcedure [dbo].[usp_GetPDFRequestByBoxReference]    Script Date: 16/10/2025 2:29:46 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[usp_GetPDFRequestByBoxReference]
	@BoxReference nvarchar(max)
AS
BEGIN

Select Request from t_PDF where Request like '% "ContainerID": "'+@BoxReference+'"%' AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive');

END
=== 
    
