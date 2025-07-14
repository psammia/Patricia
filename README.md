


using BLL;
using Microsoft.Extensions.Options;

WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();

// Add services to the container.
builder.Services.Configure<GlobalSettings>(builder.Configuration.GetSection("GlobalSettings") ?? throw new ArgumentNullException(nameof(GlobalSettings)));

builder.Services.Configure<LoyaltyResponseCode>(builder.Configuration.GetSection("ResponseCodes") ?? throw new ArgumentNullException(nameof(LoyaltyResponseCode)));

builder.Services.AddScoped(resolver =>
{
    IOptionsMonitor<GlobalSettings> optionsMonitor = resolver.GetRequiredService<IOptionsMonitor<GlobalSettings>>();
    return optionsMonitor.CurrentValue;
});


builder.Services.AddScoped(resolver =>
{
    IOptionsMonitor<LoyaltyResponseCode> optionsMonitor = resolver.GetRequiredService<IOptionsMonitor<LoyaltyResponseCode>>();
    return optionsMonitor.CurrentValue;
});

builder.Services.AddControllers();

builder.Services.AddEndpointsApiExplorer();

builder.Services.AddHttpClient("GeneralHttpClient");

builder.Services.AddScoped<Bll>();

builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
app.UseSwagger();
app.UseSwaggerUI();


app.UseCors(corsOptions => corsOptions
    .AllowAnyOrigin()
    .AllowAnyHeader()
    .AllowAnyMethod());

//app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();

    public LoyaltyController(Bll bll, 
        IOptionsMonitor<GlobalSettings> globlaSettings, 
        IOptionsMonitor<LoyaltyResponseCodes> responseCodes)

response codes returning count 0 while it should include data 


    }
  },
  "AllowedHosts": "*",
  "GlobalSettings": {
    "ConnString": "Data Source=DEV-SQL2016\\PERFAPP;Initial Catalog=Alterna.Loyalty;User ID=sasadmin;Password=sasadmin;TrustServerCertificate=True",
    "IsHttps": false,
    "IsSwaggerEnabled": true,
    "AppUsername": "AlternaLoyaltyBackend"
  },
  "ResponseCodes": {
    "ResponseCodes": [
      {
        "Code": "200",
        "Content": "0x0000",
        "Description": "Success"
      },
      {
        "Code": "422",
        "Content": "0x0002",
        "Description": "{0}"
      },
      {
        "Code": "400",
        "Content": "0x0400",
        "Description": "Bad Request {0}"
      },
      {
        "Code": "500",
        "Content": "0x0500",
        "Description": "Internal Server Error With Error Message {0}"
      }
    ]
  },

and this is my appsettings.json
