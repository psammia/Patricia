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
│   │   ├── Index.cshtml
│   │   ├── Create.cshtml
│   │   ├── Edit.cshtml
│   │   ├── Delete.cshtml
│   │   └── Details.cshtml
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

Repositories

// DatabaseHelper.cs
public class DatabaseHelper
{
    private readonly string _connectionString;

    public DatabaseHelper(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection");
    }

    public SqlConnection GetConnection()
    {
        return new SqlConnection(_connectionString);
    }
}

// CustomerRepository.cs
public class CustomerRepository
{
    private readonly DatabaseHelper _db;

    public CustomerRepository(DatabaseHelper db)
    {
        _db = db;
    }

    public List<Customer> GetAll()
    {
        var customers = new List<Customer>();
        using var conn = _db.GetConnection();
        using var cmd = new SqlCommand("sp_GetAllCustomers", conn) { CommandType = CommandType.StoredProcedure };
        conn.Open();
        using var reader = cmd.ExecuteReader();
        while (reader.Read())
        {
            customers.Add(new Customer {
                CustomerId = (int)reader["CustomerId"],
                Name = reader["Name"].ToString(),
                Email = reader["Email"].ToString()
            });
        }
        return customers;
    }

    public Customer GetById(int id)
    {
        using var conn = _db.GetConnection();
        using var cmd = new SqlCommand("sp_GetCustomerById", conn) { CommandType = CommandType.StoredProcedure };
        cmd.Parameters.AddWithValue("@CustomerId", id);
        conn.Open();
        using var reader = cmd.ExecuteReader();
        if (reader.Read())
        {
            return new Customer {
                CustomerId = (int)reader["CustomerId"],
                Name = reader["Name"].ToString(),
                Email = reader["Email"].ToString()
            };
        }
        return null;
    }

    public void Add(Customer customer)
    {
        using var conn = _db.GetConnection();
        using var cmd = new SqlCommand("sp_AddCustomer", conn) { CommandType = CommandType.StoredProcedure };
        cmd.Parameters.AddWithValue("@Name", customer.Name);
        cmd.Parameters.AddWithValue("@Email", customer.Email);
        conn.Open();
        cmd.ExecuteNonQuery();
    }

    public void Update(Customer customer)
    {
        using var conn = _db.GetConnection();
        using var cmd = new SqlCommand("sp_UpdateCustomer", conn) { CommandType = CommandType.StoredProcedure };
        cmd.Parameters.AddWithValue("@CustomerId", customer.CustomerId);
        cmd.Parameters.AddWithValue("@Name", customer.Name);
        cmd.Parameters.AddWithValue("@Email", customer.Email);
        conn.Open();
        cmd.ExecuteNonQuery();
    }

    public void Delete(int id)
    {
        using var conn = _db.GetConnection();
        using var cmd = new SqlCommand("sp_DeleteCustomer", conn) { CommandType = CommandType.StoredProcedure };
        cmd.Parameters.AddWithValue("@CustomerId", id);
        conn.Open();
        cmd.ExecuteNonQuery();
    }
}


---

Next update: full CustomerController.cs and Razor views for Index, Create, Edit, Delete, and Details. Followed by stored procedures.






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

