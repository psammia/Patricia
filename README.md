USE [Alterna_Port]
GO
/****** Object:  StoredProcedure [dbo].[usp_Cancel_Pending_Invoices_From_Branch]    Script Date: 28/04/2025 11:58:34 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

ALTER PROCEDURE [dbo].[usp_Cancel_Pending_Invoices_From_Branch] 
AS
BEGIN
	SET NOCOUNT ON;
	
	DECLARE @Invoices TABLE (
	   InvoiceRef NVARCHAR(13) PRIMARY KEY
	);

	INSERT INTO @Invoices
	SELECT InvoiceRef FROM t_Invoice
	WHERE StatusId = 6

	UPDATE t_Invoice 
	SET 
	StatusId=5,
	LastModifiedBy='AlternaSysUser',
	LastModifiedDate=GETDATE()
	WHERE StatusId = 6

	UPDATE t_Invoice_Transaction
	SET 
	StatusId=5,
	LastModifiedBy='AlternaSysUser',
	LastModifiedDate=GETDATE()
	WHERE StatusId = 6

	BEGIN --Audit Records
	INSERT INTO [dbo].[t_Invoice_Audit]
           ([InvoiceRef]
		   ,[ChannelCode]
           ,[ClientNumber]
           ,[BillId]
           ,[BillType]
           ,[BillDate]
           ,[BillLastPaymentDate]
           ,[DepartureDate]
           ,[Currency]
           ,[BillAmount]
           ,[CTOAmount]
           ,[Check]
           ,[ClientName]
           ,[ShipName]
           ,[LocalAmount]
           ,[FreshAmount]
           ,[LocalCurrency]
           ,[FreshCurrency]
           ,[PenaltyLocalAmount]
           ,[PenaltyFreshAmount]
           ,[PenaltyCTO]
           ,[CorrelationId]
           ,[CustomerId]
           ,[BbUsername]
           ,[Action]
           ,[BankAction]
           ,[PaymentDate]
           ,[StatusId]
           ,[CreatedDate]
           ,[CreatedBy]
           ,[LastModifiedDate]
           ,[LastModifiedBy])
	SELECT 
		   t_Invoice.[InvoiceRef]
		  ,[ChannelCode]
		  ,[ClientNumber]
		  ,[BillId]
		  ,[BillType]
		  ,[BillDate]
		  ,[BillLastPaymentDate]
		  ,[DepartureDate]
		  ,[Currency]
		  ,[BillAmount]
		  ,[CtoAmount]
		  ,[Check]
		  ,[ClientName]
		  ,[ShipName]
		  ,[LocalAmount]
		  ,[FreshAmount]
		  ,[LocalCurrency]
		  ,[FreshCurrency]
		  ,[PenaltyLocalAmount]
		  ,[PenaltyFreshAmount]
		  ,[PenaltyCto]
		  ,[CorrelationId]
		  ,[CustomerId]
		  ,[BbUsername]
		  ,[Action]
		  ,[BankAction]
		  ,[PaymentDate]
		  ,[StatusId]
		  ,[CreatedDate]
		  ,[CreatedBy]
		  ,[LastModifiedDate]
		  ,[LastModifiedBy]
	FROM [dbo].[t_Invoice] INNER JOIN @Invoices t ON t.InvoiceRef = [dbo].[t_Invoice].InvoiceRef
	END
END

