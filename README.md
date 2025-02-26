# Patricia
/****** Script for SelectTopNRows command from SSMS  ******/
SELECT TOP (1000) [Id]
      ,[GivenNames]
      ,[FamilyName]
      ,[AutoNextKycReviewDate],
	  sOURCE
  FROM [Alterna_KYC].[dbo].[t_AuditT24CustomerTemp]
  where id = 17 AND AutoNextKycReviewDate = @DATE

 -- 1 get from t_T24Customer Table all the customer Ids with their AutoNextKycReviewDate THat are included in table t_T24CustomerTep
 --2  Get the SOurce field from t_AuditT24CustomerTemp table for all customers in step 1 (USE THE FETCHED AutoNextKycReviewDate FROM STEP 1)
 -- 3 CHECK IF ALL SOURCE IS = ALT OR ALT_BRANCH
