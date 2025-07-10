USE [Alterna.Loyalty]
GO
/****** Object:  Table [dbo].[t_Config]    Script Date: 10/07/2025 1:41:37 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Config](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[SettingName] [nvarchar](50) NOT NULL,
	[SettingValue] [nchar](10) NOT NULL,
	[SettingDescription] [nvarchar](500) NULL,
	[IsActive] [bit] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
 CONSTRAINT [PK_t_Config] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Customer_Points]    Script Date: 10/07/2025 1:41:37 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Customer_Points](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Customer_Id] [int] NOT NULL,
	[External_Id] [nvarchar](10) NULL,
	[Total_Points] [decimal](18, 4) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
 CONSTRAINT [PK_t_Customer_Points] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Operation_Type]    Script Date: 10/07/2025 1:41:37 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Operation_Type](
	[Operation_Id] [int] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](50) NOT NULL,
	[Description] [nvarchar](250) NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
 CONSTRAINT [PK_t_Operation_Type] PRIMARY KEY CLUSTERED 
(
	[Operation_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[t_Transactions]    Script Date: 10/07/2025 1:41:37 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Transactions](
	[Transaction_Id] [int] IDENTITY(1,1) NOT NULL,
	[Customer_Id] [int] NOT NULL,
	[Card_Number] [nvarchar](50) NULL,
	[External_Id] [nvarchar](10) NULL,
	[Points] [decimal](10, 4) NULL,
	[Operation_Type_Id] [int] NOT NULL,
	[CreatedDate] [datetime] NOT NULL,
	[CreatedBy] [nvarchar](250) NOT NULL,
	[LastModifiedDate] [datetime] NOT NULL,
	[LastModifiedBy] [nvarchar](250) NOT NULL,
 CONSTRAINT [PK_t_Transactions] PRIMARY KEY CLUSTERED 
(
	[Transaction_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET IDENTITY_INSERT [dbo].[t_Config] ON 

INSERT [dbo].[t_Config] ([Id], [SettingName], [SettingValue], [SettingDescription], [IsActive], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) VALUES (1, N'ExpirationYears', N'3         ', N'Number of years before points expiration', 1, N'psammia', CAST(N'2025-07-10T00:00:00.000' AS DateTime), N'psammia', CAST(N'2025-07-10T00:00:00.000' AS DateTime))
SET IDENTITY_INSERT [dbo].[t_Config] OFF
ALTER TABLE [dbo].[t_Config] ADD  CONSTRAINT [DF_t_Config_CreatedBy]  DEFAULT (N'AlternaSystemUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Config] ADD  CONSTRAINT [DF_t_Config_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Config] ADD  CONSTRAINT [DF_t_Config_LastModifiedBy]  DEFAULT (N'AlternaSystemUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Customer_Points] ADD  CONSTRAINT [DF_t_Customer_Points_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Customer_Points] ADD  CONSTRAINT [DF_t_Customer_Points_CreatedBy]  DEFAULT (N'AlternaSystemUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Customer_Points] ADD  CONSTRAINT [DF_t_Customer_Points_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Customer_Points] ADD  CONSTRAINT [DF_t_Customer_Points_LastModifiedBy]  DEFAULT (N'AlternaSystemUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Operation_Type] ADD  CONSTRAINT [DF_t_Operation_Type_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Operation_Type] ADD  CONSTRAINT [DF_t_Operation_Type_CreatedBy]  DEFAULT (N'AlternaSystemUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Operation_Type] ADD  CONSTRAINT [DF_t_Operation_Type_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Operation_Type] ADD  CONSTRAINT [DF_t_Operation_Type_LastModifiedBy]  DEFAULT (N'AlternaSystemUser') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Transactions] ADD  CONSTRAINT [DF_t_Transactions_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Transactions] ADD  CONSTRAINT [DF_t_Transactions_CreatedBy]  DEFAULT (N'AlternaSystemUser') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Transactions] ADD  CONSTRAINT [DF_t_Transactions_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
ALTER TABLE [dbo].[t_Transactions] ADD  CONSTRAINT [DF_t_Transactions_LastModifiedBy]  DEFAULT (N'AlternaSystemUser') FOR [LastModifiedBy]
GO
