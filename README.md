CREATE TABLE Customers (
    CustomerId INT IDENTITY(1,1) PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL,
    Email NVARCHAR(100),
    Phone NVARCHAR(20)
);

CREATE TABLE Orders (
    OrderId INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT NOT NULL,
    OrderDate DATE NOT NULL,
    TotalAmount DECIMAL(18,2),
    Profit DECIMAL(18,2),
    Status NVARCHAR(20), -- e.g., Paid, Unpaid
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);

CREATE TABLE OrderItems (
    OrderItemId INT IDENTITY(1,1) PRIMARY KEY,
    OrderId INT NOT NULL,
    ProductName NVARCHAR(100),
    Quantity INT,
    Price DECIMAL(18,2),
    Cost DECIMAL(18,2),
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId)
);

CREATE TABLE Payments (
    PaymentId INT IDENTITY(1,1) PRIMARY KEY,
    OrderId INT NOT NULL,
    PaymentDate DATE,
    AmountPaid DECIMAL(18,2),
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId)
);
 

CREATE PROCEDURE sp_InsertCustomer
    @Name NVARCHAR(100),
    @Email NVARCHAR(100),
    @Phone NVARCHAR(20)
AS
BEGIN
    INSERT INTO Customers (Name, Email, Phone)
    VALUES (@Name, @Email, @Phone);
END

CREATE PROCEDURE sp_GetAllCustomers
AS
BEGIN
    SELECT * FROM Customers;
END


CREATE PROCEDURE sp_InsertOrder
    @CustomerId INT,
    @OrderDate DATE,
    @TotalAmount DECIMAL(18,2),
    @Profit DECIMAL(18,2),
    @Status NVARCHAR(20)
AS
BEGIN
    INSERT INTO Orders (CustomerId, OrderDate, TotalAmount, Profit, Status)
    VALUES (@CustomerId, @OrderDate, @TotalAmount, @Profit, @Status);
    
    SELECT SCOPE_IDENTITY() AS OrderId;
END

CREATE PROCEDURE sp_InsertOrderItem
    @OrderId INT,
    @ProductName NVARCHAR(100),
    @Quantity INT,
    @Price DECIMAL(18,2),
    @Cost DECIMAL(18,2)
AS
BEGIN
    INSERT INTO OrderItems (OrderId, ProductName, Quantity, Price, Cost)
    VALUES (@OrderId, @ProductName, @Quantity, @Price, @Cost);
END

CREATE PROCEDURE sp_InsertPayment
    @OrderId INT,
    @PaymentDate DATE,
    @AmountPaid DECIMAL(18,2)
AS
BEGIN
    INSERT INTO Payments (OrderId, PaymentDate, AmountPaid)
    VALUES (@OrderId, @PaymentDate, @AmountPaid);
END

CREATE PROCEDURE sp_GetAllOrders
AS
BEGIN
    SELECT o.OrderId, c.Name AS CustomerName, o.OrderDate, o.TotalAmount, o.Profit, o.Status
    FROM Orders o
    INNER JOIN Customers c ON o.CustomerId = c.CustomerId;
END


CREATE PROCEDURE sp_GetPaymentsSummary
AS
BEGIN
    SELECT o.OrderId, SUM(p.AmountPaid) AS TotalPaid, o.TotalAmount, (o.TotalAmount - ISNULL(SUM(p.AmountPaid), 0)) AS Remaining
    FROM Orders o
    LEFT JOIN Payments p ON o.OrderId = p.OrderId
    GROUP BY o.OrderId, o.TotalAmount;
END

CREATE PROCEDURE sp_GetCashFlow
AS
BEGIN
    SELECT SUM(AmountPaid) AS TotalCashIn FROM Payments;
END

CREATE PROCEDURE sp_GetTotalProfit
AS
BEGIN
    SELECT SUM(Profit) AS TotalProfit FROM Orders;
END


