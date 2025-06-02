System.NullReferenceException: 'Object reference not set to an instance of an object.'

        public ActionResult GetContainerToNotifyWarehouse()
        {
            String session = GetSession("ArchiveData");
            ContainerToNotifyWarehouseTableModel tableModel = new();

            GetContainerToNotifyWarehouseReq ApiReq = new()
            {
                BaseReq = new BaseRequest(HttpContext, session, true),
            };
            GetContainerToNotifyWarehouseRes resp = new();

            resp = Common.ApiCall<GetContainerToNotifyWarehouseRes>(ApiReq, "GetContainerToNotifyWarehouse");

             tableModel.ContainersToNotifyWarehouseList = resp.Resp;

            return PartialView("_GetContainerToNotifyWarehouse", tableModel);
        }

    public class ContainerToNotifyWarehouseTableModel
    {
        public List<Container> ContainersToNotifyWarehouseList { get; set; } = [];
}

    #region Container
    public partial class Container
    {
        public Int32 Id { get; set; }
        public String Code { get; set; } = String.Empty;
        public String CompanyCode { get; set; } = String.Empty;
        public String Entity { get; set; } = String.Empty;
        public String CurrentLocation { get; set; } = String.Empty;
        public String StatusCode { get; set; } = String.Empty;
        public DateTime? ArchivingDate { get; set; }
        public Boolean IsDeleted { get; set; }

        public Boolean? IsNotified { get; set; }
        public String LastModifiedBy { get; set; } = String.Empty;
        public DateTime LastModifiedDate { get; set; }
        public String CreatedBy { get; set; } = String.Empty;
        public DateTime CreatedDate { get; set; }


        // Custom Properties
        public Int32 ArchivingPeriod { get; set; }
        public DateTime? DestructionDate { get; set; }
        public List<ArchivedFile> Files { get; set; } = [];
        public String? PDF { get; set; }
        public Int32 FileCount { get; set; }
        public String SentBy { get; set; } = String.Empty;
        public String ReceivedBy { get; set; } = String.Empty;
        public DateTime? ReceivedDate { get; set; }
        public String CompanyName { get; set; } = String.Empty;
        public String ContainerType { get; set; } = String.Empty;

    }

bll
        #region GetContainerToNotifyWarehouse

        public List<Container> GetContainerToNotifyWarehouse(GetContainerToNotifyWarehouseReq getContainerToNotifyWarehouseReq)
        {
            DAL.DAL iDAL = new();
            List<Container> Retlist = [];
            OnPreEventGetContainerToNotifyWarehouse?.Invoke(ref getContainerToNotifyWarehouseReq);

            DynamicParameters param = new();
            param.Add("CompanyCodes", getContainerToNotifyWarehouseReq.BaseReq.CurrentBranch);

            Retlist = iDAL.ExecuteQuery<Container>("usp_GetContainer_Sent_TobeNotifiedbyRCA", param, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetContainerToNotifyWarehouse?.Invoke(ref Retlist, ref getContainerToNotifyWarehouseReq);
            return Retlist;
        }

        #endregion

 stored procedure

ALTER     PROCEDURE [dbo].[usp_GetContainer_Sent_TobeNotifiedbyRCA] 
(-- PARAMETER LIST
	@CompanyCodes NVARCHAR(max)
)AS
BEGIN
	SET NOCOUNT ON;

	SELECT Container.Id, 
		Container.Code,
		Container.CompanyCode,
		(select CompanyName from t_Company where Container.CompanyCode = t_Company.Code) as CompanyName,
		Container.Entity,
		[dbo].[usf_GetContainerFileType](Container.Id) As ContainerType,
		Container.CurrentLocation, 
		Container.StatusCode,
		Container.ArchivingDate,
		--container.isNotified,
		Container.LastModifiedBy,
		Container.LastModifiedDate
	FROM t_Container AS Container
		where
	container.CompanyCode IN (SELECT VALUE FROM STRING_SPLIT(@CompanyCodes, ',')) AND 
		Container.CompanyCode != '' AND
		Container.IsNotified = 0 AND
		Container.StatusCode = 'SENT'
	ORDER BY Id DESC
END


 
 
