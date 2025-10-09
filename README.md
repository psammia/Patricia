USE [Alterna.TopUp]
GO
/****** Object:  Table [dbo].[t_Transaction]    Script Date: 09/10/2025 8:25:34 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[t_Transaction](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[SessionId] [nvarchar](35) NOT NULL,
	[TransactionId] [nvarchar](40) NOT NULL,
	[OrderId] [nvarchar](40) NOT NULL,
	[CustomerId] [int] NOT NULL,
	[AccountNo] [nvarchar](16) NOT NULL,
	[Amount] [decimal](18, 4) NOT NULL,
	[Currency] [nvarchar](3) NOT NULL,
	[CardNo] [nvarchar](4) NOT NULL,
	[FirstName] [nvarchar](255) NOT NULL,
	[LastName] [nvarchar](255) NOT NULL,
	[FtRef] [nvarchar](50) NULL,
	[StatusCode] [nvarchar](50) NOT NULL,
	[CreatedBy] [nvarchar](255) NOT NULL,
	[CreatedDate] [datetime2](0) NOT NULL,
	[LastModifiedBy] [nvarchar](255) NOT NULL,
	[LastModifiedDate] [datetime2](0) NOT NULL,
 CONSTRAINT [PK_t_Transaction] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
ALTER TABLE [dbo].[t_Transaction] ADD  CONSTRAINT [DF_t_Transaction_StatusCode]  DEFAULT (N'Success') FOR [StatusCode]
GO
ALTER TABLE [dbo].[t_Transaction] ADD  CONSTRAINT [DF_t_Transaction_CreatedBy]  DEFAULT (N'AlternaSystem') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[t_Transaction] ADD  CONSTRAINT [DF_t_Transaction_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[t_Transaction] ADD  CONSTRAINT [DF_t_Transaction_LastModifiedBy]  DEFAULT (N'AlternaSystem') FOR [LastModifiedBy]
GO
ALTER TABLE [dbo].[t_Transaction] ADD  CONSTRAINT [DF_t_Transaction_LastModifiedDate]  DEFAULT (getdate()) FOR [LastModifiedDate]
GO
