CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(100) NOT NULL,
    Email NVARCHAR(100),
    Phone NVARCHAR(20)
);


CREATE TABLE Orders (
    OrderId INT PRIMARY KEY IDENTITY,
    OrderDate DATETIME NOT NULL,
    Cost DECIMAL(18,2) NOT NULL,
    Profit DECIMAL(18,2) NOT NULL,
    IsPaid BIT NOT NULL
);

CREATE TABLE CustomerOrders (
    CustomerId INT,
    OrderId INT,
    PRIMARY KEY (CustomerId, OrderId),
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId),
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId)
);

CREATE PROCEDURE UpsertCustomer
    @CustomerId INT OUTPUT,
    @Name NVARCHAR(100),
    @Email NVARCHAR(100),
    @Phone NVARCHAR(20)
AS
BEGIN
    IF EXISTS (SELECT 1 FROM Customers WHERE CustomerId = @CustomerId)
    BEGIN
        UPDATE Customers
        SET Name = @Name, Email = @Email, Phone = @Phone
        WHERE CustomerId = @CustomerId;
    END
    ELSE
    BEGIN
        INSERT INTO Customers (Name, Email, Phone)
        VALUES (@Name, @Email, @Phone);
        SET @CustomerId = SCOPE_IDENTITY();
    END
END

CREATE PROCEDURE UpsertOrder
    @OrderId INT OUTPUT,
    @OrderDate DATETIME,
    @Cost DECIMAL(18,2),
    @Profit DECIMAL(18,2),
    @IsPaid BIT
AS
BEGIN
    IF EXISTS (SELECT 1 FROM Orders WHERE OrderId = @OrderId)
    BEGIN
        UPDATE Orders
        SET OrderDate = @OrderDate, Cost = @Cost, Profit = @Profit, IsPaid = @IsPaid
        WHERE OrderId = @OrderId;
    END
    ELSE
    BEGIN
        INSERT INTO Orders (OrderDate, Cost, Profit, IsPaid)
        VALUES (@OrderDate, @Cost, @Profit, @IsPaid);
        SET @OrderId = SCOPE_IDENTITY();
    END
END

CREATE PROCEDURE GetUnpaidOrders
AS
BEGIN
    SELECT * FROM Orders WHERE IsPaid = 0;
END


CREATE PROCEDURE GetCustomersWithOrders
AS
BEGIN
    SELECT c.CustomerId, c.Name, c.Email, c.Phone,
           o.OrderId, o.OrderDate, o.Cost, o.Profit, o.IsPaid
    FROM Customers c
    INNER JOIN CustomerOrders co ON c.CustomerId = co.CustomerId
    INNER JOIN Orders o ON co.OrderId = o.OrderId;
END


