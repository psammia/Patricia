ASP.NET Core MVC ADO.NET Order Tracking System

Project Structure

OrderTrackingSystem/
├── Controllers/
│   ├── CustomerController.cs
│   ├── OrderController.cs
│   ├── OrderItemController.cs
│   └── PaymentController.cs
├── Models/
│   ├── Customer.cs
│   ├── Order.cs
│   ├── OrderItem.cs
│   └── Payment.cs
├── Repositories/
│   ├── DatabaseHelper.cs
│   ├── CustomerRepository.cs
│   ├── OrderRepository.cs
│   ├── OrderItemRepository.cs
│   └── PaymentRepository.cs
├── Views/
│   ├── Customer/
│   ├── Order/
│   ├── OrderItem/
│   └── Payment/
├── appsettings.json
├── Program.cs
└── Startup.cs


---

Program.cs

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllersWithViews();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Customer}/{action=Index}/{id?}");

app.Run();


---

appsettings.json

{
  "ConnectionStrings": {
    "DefaultConnection": "Server=YOUR_SERVER_NAME;Database=OrderTrackingDB;Trusted_Connection=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}


---

Models

// Customer.cs
public class Customer {
    public int CustomerId { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

// Order.cs
public class Order {
    public int OrderId { get; set; }
    public int CustomerId { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal TotalCost { get; set; }
    public decimal Profit { get; set; }
    public bool IsPaid { get; set; }
}

// OrderItem.cs
public class OrderItem {
    public int OrderItemId { get; set; }
    public int OrderId { get; set; }
    public string Description { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal CostPrice { get; set; }
}

// Payment.cs
public class Payment {
    public int PaymentId { get; set; }
    public int OrderId { get; set; }
    public decimal AmountPaid { get; set; }
    public DateTime PaymentDate { get; set; }
}


---

Would you like me to continue by adding full Customer controller, views, and repository with ADO.NET using stored procedures?

