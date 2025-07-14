namespace BLL;

public class LoyaltyResponseCodes
{
    public List<LoyaltyResponseCode> ResponseCodes { get; set; } = [];
}

public class LoyaltyResponseCode
{
    public String Code { get; set; } = String.Empty;
    public String Content { get; set; } = String.Empty;
    public String Description { get; set; } = String.Empty;
}
i already have the types 
using Microsoft.AspNetCore.Mvc;
using System.Text;
using BLL;
using Microsoft.Extensions.Options;
using static NLog.NLogUtil;
using static BLL.CustomExceptions;
using static BLL.DataGuard;

namespace Alterna_Loyalty.Controllers;

[ApiController]
[Route("api/[controller]")]
public class LoyaltyController : ControllerBase
{
    private readonly Bll _bll;
    private readonly GlobalSettings _globalSettings;
    private readonly Dictionary<String, LoyaltyResponseCode> _reponseCodesDictionary = [];

    public LoyaltyController(Bll bll, 
        IOptionsMonitor<GlobalSettings> globlaSettings, 
        IOptionsMonitor<LoyaltyResponseCodes> responseCodes)



the prob;lem is here the resonscodes comeback with count 0
