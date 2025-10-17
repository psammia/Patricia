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

Id	Code	Description	Category	CreatedBy	CreatedDate	LastModifiedBy	LastModifiedDate
1	PENDING	Container created	CONTAINER	ArchivingInit	2024-06-10 10:54:26.733	ArchivingInit	2024-06-10 10:54:26.733
2	SENTFORVAL	Container sent for validation by the  RCA	CONTAINER	ArchivingInit	2024-06-10 10:54:26.740	ArchivingInit	2024-06-10 10:54:26.740
3	VALIDATED	Container status validate by the DA	CONTAINER	ArchivingInit	2024-06-10 10:54:26.740	ArchivingInit	2024-06-10 10:54:26.740
4	SENT	Container sent to the warehouse	CONTAINER	ArchivingInit	2024-06-10 10:54:26.747	ArchivingInit	2024-06-10 10:54:26.747
5	RECEIVED	Container received to the warehouse	CONTAINER	ArchivingInit	2024-06-10 10:54:26.750	ArchivingInit	2024-06-10 10:54:26.750
6	TOBEDESTR	Container reaching the expiry date, waiting to be destroyed	CONTAINER	ArchivingInit	2024-06-10 10:54:26.753	ArchivingInit	2024-06-10 10:54:26.753
7	DESTROYED	Container destroyed	CONTAINER	ArchivingInit	2024-06-10 10:54:26.757	ArchivingInit	2024-06-10 10:54:26.757


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
   Composed from BackEnd and FrontEnd, using MVC.
   Back:  is composed from Archiving.Controller + BLL + CustomCode class for events.
   Front is composed from FilesController

 == FilesController.cs
        public ActionResult ReDownloadSendPDF(String boxReference)
        {
            DownloadPDFModel model = new();
            DownloadPDFRes downloadPDFRes = Common.ApiCall<DownloadPDFRes>(new DownloadPDFReq()
            {
                BaseReq = new BaseRequest(HttpContext,GetSession("ArchiveData")),
                ContainerID = boxReference
            }, "DownloadPDF");

            if (downloadPDFRes.Resp is null || downloadPDFRes.Resp.Length == 0)
            {
                HttpContext.Session.SetString("CorrelationId", downloadPDFRes.WebResp.CorrelationId);
                HttpContext.Session.SetString("ErrorMessage", "Invalid PDF");

                throw new ErrorHandler(new ErrorModel() { ErrorCorrelationId = downloadPDFRes.WebResp.CorrelationId, ErrorMessage = "Invalid PDF" });
            }


            String PDF = downloadPDFRes.Resp ?? String.Empty;

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


=== BLL.cs
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

===Back === ArchivingController.cs

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
		    #region Edit Container Status Prevent Adition
    public partial class EditContainerStatusReq
    {
        public String? PDF { get; set; }
    }
	    public partial class EditContainerStatusRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required EditContainerStatusReq Req { get; set; }
        public Container? Resp { get; set; }
    }
    
		    public partial class DownloadPDFReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String ContainerID { get; set; }
        public ArchivingDocumentType? DocumentType { get; set; }
    }
	    public partial class DownloadPDFRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required DownloadPDFReq Req { get; set; }
        public String? Resp { get; set; }
    }
====Controller.cs

		    public class DownloadPDFModel
    {
        public List<FileType> FileTypeList { get; set; } = [];
        public List<SelectListItem> EntityList { get; set; } = [];

    }
	    public partial class DownloadPDFRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public DownloadPDFReq? Req { get; set; }
        public String Resp { get; set; } = String.Empty;
    }

	=== BLL.cs ====
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

		
		    public partial class DownloadPDFRes
    {
        public BaseResponse WebResp { get; set; } = new BaseResponse();
        public required DownloadPDFReq Req { get; set; }
        public String? Resp { get; set; }
    }

	    public partial class DownloadPDFReq
    {
        public BaseRequest BaseReq { get; set; } = new BaseRequest();
        public required String ContainerID { get; set; }
        public ArchivingDocumentType? DocumentType { get; set; }
    }


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

=== CustomerCode.cs
        private void BLL_OnPreEventEditContainerStatus(ref EditContainerStatusReq editContainerStatusReq)
        {
            if (editContainerStatusReq.StatusCode.Equals(ContainerStatusCode.SENT.ToString()) && !String.IsNullOrEmpty(editContainerStatusReq.Code))
            {
                editContainerStatusReq.PDF = String.Empty;
                String Entity = GetActiveEntity(editContainerStatusReq.HoldingEntityCode);

                GetContainerFilesReq getContainerFilesReq = new()
                {
                    BaseReq = editContainerStatusReq.BaseReq,
                    ContainerId = editContainerStatusReq.ContainerId
                };

                List<ArchivedFile> files = [];
                files = GetContainerFiles(getContainerFilesReq).Files;
                if (files.Count > 0 || editContainerStatusReq.Code is not null)
                {
                    Boolean Unlimited = false;
                    DateTime ArchivePeriod = DateTime.Now;
                    ArchivePeriod = ArchivePeriod.AddYears(files[0].ArchivingPeriod);

                    if (files[0].ArchivingPeriod == -1)
                    {
                        Unlimited = true;
                    }

                    if (files[0].CustomerId != null)
                    {
                        CustomerDocRequest customerDocRequest = new()
                        {
                            DestructionDate = Unlimited? "Unlimited":$"{ArchivePeriod:dd/MM/yyyy}",
                            ContainerID = editContainerStatusReq.Code!,
                            Entity = Entity,
                            User = editContainerStatusReq.BaseReq.CurrentUser!,
                            CustomerFiles = [],
                            CreationDate = $"{DateTime.Now:dd/MM/yyyy}"
                        };

                        Dictionary<String,List<String?>> fileDict=[];
                        foreach (ArchivedFile item in files)
                        {
                            if (!fileDict.ContainsKey(item.Name))
                            {
                                fileDict.Add(item.Name, [item.CustomerId!.ToString()]);
                            }
                            else
                            {
                                fileDict[item.Name].Add(item.CustomerId!.ToString());
                            }
                        }
                        foreach (KeyValuePair<String, List<String?>> DictEntry in fileDict)
                        {
                            customerDocRequest.CustomerFiles.Add(new() { DocumentType = DictEntry.Key, Id = DictEntry.Value! });
                        }
                        try
                        {
                            String data=JsonConvert.SerializeObject(customerDocRequest);
                            HttpContent content = new StringContent(data,Encoding.UTF8,"application/json");
                            HttpClient client = new();
                            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"]??throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                            Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}GenerateCustomerDocPDFForArchive", content);

                            Request.Wait();
                            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                            responseString.Wait();
                            editContainerStatusReq.PDF = responseString.Result;
                        }
                        catch (Exception ex)
                        {
                            throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
                        }

                    }
                    else if (files[0].CompanyCode.StartsWith("LB"))
                    {
                        BranchDocRequest branchDocRequest=new()
                        {
                            DestructionDate= Unlimited? "Unlimited":$"{ArchivePeriod:dd/MM/yyyy}",
                            ContainerID=editContainerStatusReq.Code!,
                            Entity=Entity,
                            User =editContainerStatusReq.BaseReq.CurrentUser!,
                            BranchFiles=[],
                            CreationDate = $"{DateTime.Now:dd/MM/yyyy}"
                        };
                        foreach (ArchivedFile item in files)
                        {
                            branchDocRequest.BranchFiles.Add(new()
                            {
                                DocumentType = item.Name,
                                FromDate = $"{item.FromDate:dd-MM-yyyy}",
                                ToDate = $"{item.ToDate:dd-MM-yyyy}"
                            });
                        }
                        try
                        {
                            String data=JsonConvert.SerializeObject(branchDocRequest);
                            HttpContent content = new StringContent(data,Encoding.UTF8,"application/json");
                            HttpClient client = new();
                            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"]??throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                            Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}GenerateBranchDocPDFForArchive", content);

                            Request.Wait();
                            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                            responseString.Wait();
                            editContainerStatusReq.PDF = responseString.Result;
                        }
                        catch (Exception ex)
                        {
                            throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
                        }

                    }
                    else if (files[0].CompanyCode.StartsWith("ET"))
                    {
                        EntityDocRequest entityDocRequest = new()
                        {
                            DestructionDate = Unlimited? "Unlimited":$"{ArchivePeriod:dd/MM/yyyy}",
                            ContainerID = editContainerStatusReq.Code!,
                            Entity = files[0].CompanyCode,
                            User = editContainerStatusReq.BaseReq.CurrentUser!,
                            EntityFiles = [],
                            CreationDate = $"{DateTime.Now:dd/MM/yyyy}"
                        };
                        foreach (ArchivedFile item in files)
                        {
                            entityDocRequest.EntityFiles.Add(new()
                            {
                                DocumentType = item.Name,
                                DocumentDescription = item.AdditionalInfo ?? String.Empty
                            });
                        }
                        try
                        {
                            String data = JsonConvert.SerializeObject(entityDocRequest);
                            HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
                            HttpClient client = new();
                            String PDFRequestBase = ConfigurationManager.AppSettings["PDFService"] ?? throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

                            Task<HttpResponseMessage> Request = client.PostAsync($"{PDFRequestBase}GenerateEntityDocPDFForArchive", content);

                            Request.Wait();
                            Task<String> responseString = Request.Result.Content.ReadAsStringAsync();
                            responseString.Wait();
                            editContainerStatusReq.PDF = responseString.Result;
                        }
                        catch (Exception ex)
                        {
                            throw new SGBLInternalServerException("PDF Creation Failed Please Contact Support", ex.InnerException!);
                        }
                    }
                }
                else
                {
                    throw new SGBLBadRequestException($"The Container With Sequence {editContainerStatusReq.Code} Does Not Contain Any Files\n\rPlease Contact Support");
                }
                if (String.IsNullOrEmpty(editContainerStatusReq.PDF))
                {
                    throw new SGBLInternalServerException("PDF Service Malfunction");
                }
            }
        }

        private void BLL_OnPostEventEditContainerStatus(ref Container Container, ref EditContainerStatusReq editContainerStatusReq)
        {
            if (!String.IsNullOrEmpty(editContainerStatusReq.PDF))
            {
                Container.PDF = editContainerStatusReq.PDF;
            }
        }

		=== Partial View
		@using Alterna.Archive.Core.Global
@using Alterna.Archive.Core.Models

@model Alterna.Archive.Core.Models.TableModel.EntityFilesTableModel

<table id="TblentityFilesTable" class="table table-striped table-bordered" style="width:100%;">
    <thead>
        <tr>
            <th></th>
            <th>Box</th>
            <th>File Type</th>
            <th>File Info</th>
            <th>Period</th>
            <th>Inputter</th>
            <th>Archiving Date</th>
            <th>Status</th>

        </tr>
    </thead>
    <tbody>
        @if (Model.EntityFilesList.Count > 0)
        {
            foreach (ArchivedFile file in Model.EntityFilesList)
            {
                String iconId = "containerDetails";
                String containerCodes = "";
                DateTime? archivingDate = null;

                foreach (Container container in file.FileContainers)
                {
                    iconId += container.Id + "-";
                    containerCodes += container.Code + "-";
                    archivingDate = container.ArchivingDate;
                }

                iconId = iconId.Remove(iconId.Length - 1);
                containerCodes = containerCodes.Remove(containerCodes.Length - 1);

                <tr>
                    @if (
                   file.Status == Const.ContainerStatusCode.SENT.ToString() ||
                   file.Status == Const.ContainerStatusCode.RECEIVED.ToString() ||
                   file.Status == Const.ContainerStatusCode.TOBEDESTR.ToString() ||
                   file.Status == Const.ContainerStatusCode.DESTROYED.ToString()
                   )
                    {
                        <td class="text-center">
                            <i id="@iconId" class="fa-solid fa-magnifying-glass icon-detail" title="More Details" style="cursor: pointer;" onclick="openDetails('@iconId')"></i>&nbsp;&nbsp
                            <i id="@containerCodes" class="fa-solid fa-download" title="Re-Download PDF file" style="cursor: pointer;" onclick="downloadPDF('@containerCodes')"></i>
                        </td>
                    }
                    else
                    {
                        <td class="text-center">
                            <i id="@iconId" class="fa-solid fa-magnifying-glass icon-detail" title="More Details" style="cursor: pointer;" onclick="openDetails('@iconId')"></i>
                            &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
                        </td>
                    }

                    <td>@containerCodes</td>
                    <td>@file.Name</td>
                    <td>@file.AdditionalInfo</td>

                    @if (file.ArchivingPeriod == -1)
                    {
                        <td>Unlimited</td>
                    }
                    else
                    {
                        <td>@file.ArchivingPeriod</td>
                    }

                    <td>@file.CreatedBy</td>

                    @if (archivingDate.HasValue)
                    {
                        <td>@archivingDate.Value.ToString("dd/MM/yyyy")</td>
                    }
                    else
                    {
                        <td></td>
                    }

                    <td>@file.Status</td>
                </tr>
            }
        }
    </tbody>
</table>

<button id="BtnSearchAgain" type="button" class="btn btn-primary" style="margin-top: 6px" onclick="SearchAgain()">Search Again</button>

<script>
    $(document).ready(() => {

        $("#TblentityFilesTable").DataTable(
            {
                pagingType: 'full_numbers',
                responsive: true
            });

    })

    function openDetails(containerId) {
        let containerIds = containerId.replace("containerDetails", "").split("-");

        $('#ContainerDetailsContainer').html("");


        containerIds.forEach((containerId) => {

            $.ajax({
                type: 'POST',
                url: '/Files/GetEntityContainerFiles/',
                data: {
                    ContainerId: parseInt(containerId),
                },
                dataType: 'html',
                success: function (response) {
                    $('#ContainerDetailsContainer').append('<div id="ContainerDetails' + containerId + '"></div>');
                    $('#ContainerDetails' + containerId).html(response);
                    $('#TableDisplay').hide();
                },
                error: function (xhr) {
                    $('#MainRenderLocation').html(xhr.responseText);
                }
            });
        });
    }

    function SearchAgain() {
        $('#ContainerDetailsContainer').html("");
        $('#TableDisplay').html("");
        $("#EntityFilesFilterOptions").show();
    }

    function downloadPDF(boxRef) {
        window.open('@Url.Action("ReDownloadSendPDF", "Files")?boxReference=' + boxRef, '_blank').focus();
    }
</script>

	
2) Project No.2 Name: PdfGenerator
   Composed from Base Controller and BLL.cs

   ==BaseController.cs

       [HttpPost]
    [Route("GenerateEntityDocPDFForArchive")]
    public string GenerateEntityDocPDFForArchive(EntityDocRequest requ)
    {
        BLL.BLL ArchiveBll = new();
        var myByteArray = ArchiveBll.GenerateEntityDocPDFForArchive(requ);
        StringBuilder sb = new(myByteArray.Length * 2);
        foreach (var b in myByteArray) sb.AppendFormat("{0:x2}", b);
        return sb.ToString();
    }
   
    [HttpPost]
    [Route("RedownloadDocPDFForArchive")]
    public string RedownloadDocPDFForArchive(RedownloadDocPDFForArchiveRequest requ)
    {
        BLL.BLL ArchiveBll = new();
        var myByteArray = ArchiveBll.RedownloadDocPDFForArchive(requ);
        StringBuilder sb = new(myByteArray.Length * 2);
        foreach (var b in myByteArray) sb.AppendFormat("{0:x2}", b);
        return sb.ToString();
    }
   
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


	  ALTER PROCEDURE [dbo].[usp_EditContainerStatus] (
    -- PARAMETER LIST
    @ContainerId INT,
    @StatusCode NVARCHAR(10),
    @HoldingEntityCode NVARCHAR(max),
    @User NVARCHAR(250)
  ) AS BEGIN
SET
  NOCOUNT ON;
 DECLARE @FirstActiveBranch nvarchar(9);

SET
  @FirstActiveBranch = COALESCE(
    (
      SELECT
        TOP 1 Owner
      FROM
        t_Sequence
      WHERE
        IsActive = 1
        AND Owner IN (
          SELECT
            value
          FROM
            string_split(@HoldingEntityCode, ',')
        )
    ),
    'ERROR'
  );
  
UPDATE
  t_ContainerStatus
SET
  isCurrentStatus = 0
WHERE
  ContainerId = @ContainerId
UPDATE
  t_Container
SET
  StatusCode = @StatusCode,
  LastModifiedBy = @User,
  LastModifiedDate = GETDATE()
WHERE
  Id = @ContainerId
INSERT INTO
  t_ContainerStatus (
    ContainerId,
    StatusCode,
    HoldingEntityCode,
    isCurrentStatus,
    CreatedBy,
    LastModifiedBy
  )
VALUES
  (
    @ContainerId,
    @StatusCode,
    @FirstActiveBranch,
    1,
    @User,
    @User
  )
SELECT
  Id,
  CompanyCode,
  CurrentLocation,
  StatusCode,
  isDeleted
FROM
  t_Container
WHERE
  Id = @ContainerId
END
