using Alterna.Archive.Core.Global;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.CookiePolicy;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Options;
using Newtonsoft.Json.Serialization;

internal class Program
{
    private static void Main(String[] args)
    {
        String? isRin = System.Configuration.ConfigurationManager.AppSettings["isRinActive"];

        WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

        builder.Services.AddControllers().AddNewtonsoftJson(options =>
        {
            options.SerializerSettings.ContractResolver = new DefaultContractResolver();
        });
        builder.Services.AddControllers().AddJsonOptions(options =>
        {
            options.JsonSerializerOptions.PropertyNamingPolicy = null;
        });
        // Add services to the container.
        if (isRin == "true")
        {
            builder.Logging.AddRinLogger();
            builder.Services.AddRin();
            builder.Services.AddControllersWithViews().AddRinMvcSupport();
        }
        else
        {
            builder.Services.AddControllersWithViews();
        }

        builder.Services.AddDistributedMemoryCache();
        builder.Services.AddSession();
#if !DEBUG
        if (!String.IsNullOrEmpty(System.Configuration.ConfigurationManager.AppSettings["DomainConfig"]))
        {
            builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie(options =>
            {
                options.Cookie.Domain = System.Configuration.ConfigurationManager.AppSettings["DomainConfig"];
                options.Cookie.SameSite = SameSiteMode.Lax;
                options.Cookie.SecurePolicy = CookieSecurePolicy.SameAsRequest;
                options.Cookie.HttpOnly = true;
                options.Cookie.Path = "/";
            });
        }
#endif

        builder.Services.AddMvcCore().AddRazorViewEngine(opt =>
        {
            opt.ViewLocationFormats.Add("/Views/{1}/{0}.cshtml");
            opt.ViewLocationFormats.Add("/Views/{1}/PartialViews/{0}.cshtml");
            opt.ViewLocationFormats.Add("/Views/Shared/{0}.cshtml");
        });


        WebApplication app = builder.Build();

        // Configure the HTTP request pipeline.
        if (!app.Environment.IsDevelopment())
        {
            app.UseExceptionHandler("/Home/Error");
        }
        else
        {
            app.UseDeveloperExceptionPage();
        }

        if (isRin == "true")
        {
            app.UseRin();

            app.UseRinDiagnosticsHandler();
        }

        app.UseStaticFiles();

        app.UseRouting();

        app.UseAuthorization();
        app.UseCookiePolicy();
        app.UseSession();
        app.MapControllerRoute(
            name: "default",
            pattern: "{controller=Home}/{action=Index}/{Param1?}/{Param2?}");
        app.UseStatusCodePagesWithReExecute("/Home/Error");
        app.Run();
    }
}
