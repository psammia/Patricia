USE [Alterna.OnBoarding]
GO
/****** Object:  Table [dbo].[t_App_Files]    Script Date: 30/09/2025 2:42:20 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_App_Files](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[App_Id] [bigint] NOT NULL,
	[File_Name] [nvarchar](255) NOT NULL,
	[File_Type] [nvarchar](100) NOT NULL,
	[File_Size] [bigint] NOT NULL,
	[File_Data] [varbinary](max) NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime2](7) NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime2](7) NOT NULL,
 CONSTRAINT [PK_t_App_Files] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Application]    Script Date: 30/09/2025 2:42:21 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Application](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[External_Id] [nvarchar](10) NOT NULL,
	[CorrelationId] [nvarchar](250) NOT NULL,
	[StatusId] [int] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime2](0) NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime2](0) NOT NULL,
 CONSTRAINT [PK_t_Application] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Status]    Script Date: 30/09/2025 2:42:21 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Status](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[StatusCode] [nvarchar](50) NOT NULL,
	[StatusDescription] [nvarchar](250) NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime2](0) NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime2](7) NOT NULL,
 CONSTRAINT [PK_t_Status] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
ALTER TABLE [dbo].[t_App_Files] ADD  CONSTRAINT [DF_t_App_Files_CreatedBy]  DEFAULT (N'AlternaSysUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_App_Files] ADD  CONSTRAINT [DF_t_App_Files_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_App_Files] ADD  CONSTRAINT [DF_t_App_Files_LastModifiedBy]  DEFAULT (N'AlternaSysUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_App_Files] ADD  CONSTRAINT [DF_t_App_Files_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Application] ADD  CONSTRAINT [DF_t_Application_CreatedBy]  DEFAULT (N'AlternaSysUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Application] ADD  CONSTRAINT [DF_t_Application_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Application] ADD  CONSTRAINT [DF_t_Application_LastModifiedBy]  DEFAULT (N'AlternaSysUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Application] ADD  CONSTRAINT [DF_t_Application_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Status] ADD  CONSTRAINT [DF_t_Status_CreatedBy]  DEFAULT (N'AlternaSysUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Status] ADD  CONSTRAINT [DF_t_Status_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Status] ADD  CONSTRAINT [DF_t_Status_LastModifiedBy]  DEFAULT (N'AlternaSysUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Status] ADD  CONSTRAINT [DF_t_Status_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
