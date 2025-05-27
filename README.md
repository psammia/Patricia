USE [OrderTracking]
GO
/****** Object:  User [sgbl\scomr2admin]    Script Date: 27/05/2025 4:08:44 PM ******/
CREATE USER [sgbl\scomr2admin] FOR LOGIN [sgbl\scomr2admin] WITH DEFAULT_SCHEMA=[dbo]
GO
/****** Object:  Table [dbo].[CustomerOrders]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[CustomerOrders](
	[CustomerId] [int] NOT NULL,
	[OrderId] [int] NOT NULL,
	[NoOfProductperCustomer] [int] NULL,
	[IsPaid] [bit] NOT NULL,
	[Amount] [decimal](18, 0) NULL,
	[CustomerName] [nchar](100) NULL,
 CONSTRAINT [PK__Customer__489761644479D49F] PRIMARY KEY CLUSTERED 
(
	[CustomerId] ASC,
	[OrderId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[Customers]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[Customers](
	[CustomerId] [int] IDENTITY(1,1) NOT NULL,
	[Name] [nvarchar](100) NOT NULL,
	[Phone] [nvarchar](20) NULL,
	[Address] [nvarchar](max) NULL,
PRIMARY KEY CLUSTERED 
(
	[CustomerId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[Orders]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[Orders](
	[OrderId] [int] IDENTITY(1,1) NOT NULL,
	[OrderDate] [datetime] NOT NULL,
	[Profit] [decimal](18, 2) NULL,
	[NoOfProduct] [int] NULL,
	[TotalAmount] [decimal](18, 0) NULL,
	[StatusCode] [nvarchar](100) NULL,
 CONSTRAINT [PK__Orders__C3905BCF9BF4991D] PRIMARY KEY CLUSTERED 
(
	[OrderId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[Status]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[Status](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[StatusCode] [nvarchar](50) NULL,
	[StatusDescription] [nvarchar](250) NULL,
 CONSTRAINT [PK_Status] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
ALTER TABLE [dbo].[CustomerOrders]  WITH CHECK ADD  CONSTRAINT [FK__CustomerO__Custo__2C3393D0] FOREIGN KEY([CustomerId])
REFERENCES [dbo].[Customers] ([CustomerId])
GO
ALTER TABLE [dbo].[CustomerOrders] CHECK CONSTRAINT [FK__CustomerO__Custo__2C3393D0]
GO
ALTER TABLE [dbo].[CustomerOrders]  WITH CHECK ADD  CONSTRAINT [FK__CustomerO__Order__2D27B809] FOREIGN KEY([OrderId])
REFERENCES [dbo].[Orders] ([OrderId])
GO
ALTER TABLE [dbo].[CustomerOrders] CHECK CONSTRAINT [FK__CustomerO__Order__2D27B809]
GO
/****** Object:  StoredProcedure [dbo].[GetCustomersWithOrders]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[GetCustomersWithOrders]
AS
BEGIN
    SELECT c.CustomerId, c.Name, c.Address, c.Phone,
           o.OrderId, o.OrderDate, o.Cost, o.Profit, o.NoOfProduct, o.TotalAmount, o.StatusCode
    FROM Customers c
    INNER JOIN CustomerOrder co ON c.CustomerId = co.CustomerId
    INNER JOIN Orders o ON co.OrderId = o.OrderId;
END
GO
/****** Object:  StoredProcedure [dbo].[UpsertCustomer]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[UpsertCustomer]
    @CustomerId INT OUTPUT,
    @Name NVARCHAR(100),
    @Address NVARCHAR(100),
    @Phone NVARCHAR(20)
AS
BEGIN
    IF EXISTS (SELECT 1 FROM Customers WHERE CustomerId = @CustomerId)
    BEGIN
        UPDATE Customers
        SET Name = @Name, Address = @Address, Phone = @Phone
        WHERE CustomerId = @CustomerId;
    END
    ELSE
    BEGIN
        INSERT INTO Customers (Name, Address, Phone)
        VALUES (@Name, @Address, @Phone);
        SET @CustomerId = SCOPE_IDENTITY();
    END
END
GO
/****** Object:  StoredProcedure [dbo].[UpsertOrder]    Script Date: 27/05/2025 4:08:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[UpsertOrder]
    @OrderId INT OUTPUT,
    @OrderDate DATETIME,
    @Cost DECIMAL(18,2),
    @Profit DECIMAL(18,2),
	@NoOfProduct INT,
	@TotalAmount DECIMAL(18,2),
    @StatusCode NVARCHAR(50)
AS
BEGIN
    IF EXISTS (SELECT 1 FROM Orders WHERE OrderId = @OrderId)
    BEGIN
        UPDATE Orders
        SET OrderDate = @OrderDate, Cost = @Cost, Profit = @Profit,NoOfProduct = @NoOfProduct, TotalAmount = @TotalAmount, StatusCode = @StatusCode
        WHERE OrderId = @OrderId;
    END
    ELSE
    BEGIN
        INSERT INTO Orders (OrderDate, Cost, Profit,NoOfProduct,TotalAmount, StatusCode)
        VALUES (@OrderDate, @Cost, @Profit,@NoOfProduct,@TotalAmount,@StatusCode);
        SET @OrderId = SCOPE_IDENTITY();
    END
END
GO
