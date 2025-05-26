@model OrdersTracking.Models.Order

@{
    ViewData["Title"] = "Create Order";
    var customers = ViewBag.Customers as List<Customer> ?? new List<Customer>();
    var statuses = ViewBag.Statuses as List<Status> ?? new List<Status>();
    var customerOptionsHtml = new System.Text.StringBuilder();
   
    foreach (var customer in customers)
    {
        customerOptionsHtml.Append($"<option value=\"{customer.Value}\">{customer.Text}</option>");
    }
}
Severity	Code	Description	Project	File	Line	Suppression State
Error	MSB3021	Unable to copy file "D:\@Workspace\deve-repo\OrdersTracking\obj\Debug\net8.0\apphost.exe" to "bin\Debug\net8.0\OrdersTracking.exe". The process cannot access the file 'bin\Debug\net8.0\OrdersTracking.exe' because it is being used by another process.	OrdersTracking	C:\Program Files\Microsoft Visual Studio\2022\Enterprise\MSBuild\Current\Bin\amd64\Microsoft.Common.CurrentVersion.targets	5198	
Error	MSB3027	Could not copy "D:\@Workspace\deve-repo\OrdersTracking\obj\Debug\net8.0\apphost.exe" to "bin\Debug\net8.0\OrdersTracking.exe". Exceeded retry count of 10. Failed. The file is locked by: "OrdersTracking (9488)"	OrdersTracking	C:\Program Files\Microsoft Visual Studio\2022\Enterprise\MSBuild\Current\Bin\amd64\Microsoft.Common.CurrentVersion.targets	5198	
Error (active)	CS1503	Argument 1: cannot convert from 'method group' to 'object?'	OrdersTracking	D:\@Workspace\deve-repo\OrdersTracking\Views\Order\Create.cshtml	11	
Error (active)	CS1061	'Customer' does not contain a definition for 'Text' and no accessible extension method 'Text' accepting a first argument of type 'Customer' could be found (are you missing a using directive or an assembly reference?)	OrdersTracking	D:\@Workspace\deve-repo\OrdersTracking\Views\Order\Create.cshtml	11	



