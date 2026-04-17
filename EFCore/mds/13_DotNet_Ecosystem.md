# .NET Ecosystem Tools

---

## Central Package Management (CPM)

CPM centralizes NuGet package versions across multiple projects in a solution, ensuring consistency and simplifying updates.

### Directory.Packages.props

Create this file in solution root to manage versions centrally:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- EF Core -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0" />
    
    <!-- Testing -->
    <PackageVersion Include="xunit" Version="2.6.2" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.5.4" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    
    <!-- Other -->
    <PackageVersion Include="AutoMapper" Version="12.0.1" />
    <PackageVersion Include="FluentValidation" Version="11.8.1" />
    
    <!-- Version override available -->
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>
</Project>
```

### Project File Usage

Remove version attributes from individual project files:

```xml
<!-- Before CPM -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />

<!-- After CPM -->
<PackageReference Include="Microsoft.EntityFrameworkCore" />
```

### Version Overrides

Individual projects can override central versions when necessary:

```xml
<PackageReference Include="Newtonsoft.Json" VersionOverride="12.0.3" />
```

### Benefits

- **Consistency:** Same version across all projects
- **Simplicity:** Update version in one location
- **Visibility:** Central view of all dependencies
- **Security:** Easier to audit and update vulnerable packages

### Transitive Pinning

Pin transitive dependencies to specific versions:

```xml
<ItemGroup>
  <PackageVersion Include="System.Text.Json" Version="8.0.0" />
  <PackageVersion Include="System.Net.Http" Version="4.3.4" />
</ItemGroup>
```

---

## Bogus Library

Bogus generates realistic fake data for testing and development using fluent syntax and built-in datasets.

### Installation

```xml
<PackageReference Include="Bogus" Version="35.0.1" />
```

### Basic Usage

```csharp
using Bogus;

// Simple generator
var faker = new Faker();
var name = faker.Name.FullName();
var email = faker.Internet.Email();
var company = faker.Company.CompanyName();
```

### Entity Generation

```csharp
public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public DateTime DateOfBirth { get; set; }
    public string Phone { get; set; }
    public Address Address { get; set; }
    public List<Order> Orders { get; set; }
}

public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string ZipCode { get; set; }
    public string Country { get; set; }
}
```

### Faker Rules

```csharp
var customerFaker = new Faker<Customer>("en") // Locale
    .RuleFor(c => c.Id, f => f.IndexFaker + 1)
    .RuleFor(c => c.FirstName, f => f.Name.FirstName())
    .RuleFor(c => c.LastName, f => f.Name.LastName())
    .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.FirstName, c.LastName))
    .RuleFor(c => c.DateOfBirth, f => f.Date.Past(30, DateTime.Now.AddYears(-18)))
    .RuleFor(c => c.Phone, f => f.Phone.PhoneNumber("(###) ###-####"))
    .RuleFor(c => c.Address, f => addressFaker.Generate());

var addressFaker = new Faker<Address>()
    .RuleFor(a => a.Street, f => f.Address.StreetAddress())
    .RuleFor(a => a.City, f => f.Address.City())
    .RuleFor(a => a.ZipCode, f => f.Address.ZipCode())
    .RuleFor(a => a.Country, f => f.Address.Country());

// Generate single
var customer = customerFaker.Generate();

// Generate collection
var customers = customerFaker.Generate(100);
```

### Advanced Features

```csharp
// Conditional rules
var productFaker = new Faker<Product>()
    .RuleFor(p => p.Name, f => f.Commerce.ProductName())
    .RuleFor(p => p.Price, f => f.Random.Decimal(10, 1000))
    .RuleFor(p => p.IsOnSale, f => f.Random.Bool())
    .RuleFor(p => p.SalePrice, (f, p) => p.IsOnSale ? p.Price * 0.8m : null);

// Fin dataset for financial data
var amount = faker.Finance.Amount(100, 10000);
var currency = faker.Finance.Currency().Code;
var creditCard = faker.Finance.CreditCardNumber();

// Commerce dataset
var product = faker.Commerce.Product();
var color = faker.Commerce.Color();
var department = faker.Commerce.Department();
var price = faker.Commerce.Price();

// Lorem dataset
var paragraph = faker.Lorem.Paragraph();
var sentences = faker.Lorem.Sentences(3);
```

### Seeded Generation (Deterministic)

```csharp
// Same seed produces same data across runs
var seededFaker = new Faker<Customer>()
    .UseSeed(12345)
    .RuleFor(c => c.Id, f => f.IndexFaker)
    .RuleFor(c => c.Name, f => f.Name.FullName());

// Always generates same customers
var customers = seededFaker.Generate(10);
```

### Integration with EF Core

```csharp
public static class SeedDataExtensions
{
    public static async Task SeedDatabaseAsync(this ApplicationDbContext context)
    {
        if (await context.Customers.AnyAsync()) return;

        var customers = new Faker<Customer>()
            .RuleFor(c => c.Name, f => f.Name.FullName())
            .RuleFor(c => c.Email, f => f.Internet.Email())
            .Generate(50);

        await context.Customers.AddRangeAsync(customers);
        await context.SaveChangesAsync();
    }
}
```

### Dataset Categories

| Category | Examples |
|----------|----------|
| Name | FirstName, LastName, FullName, Prefix, Suffix |
| Address | StreetAddress, City, State, ZipCode, Country |
| Internet | Email, DomainName, Ip, Mac, Password, Url |
| Phone | PhoneNumber |
| Commerce | ProductName, Color, Department, Price, ProductAdjective |
| Finance | AccountNumber, CreditCardNumber, Currency, Amount, BitcoinAddress |
| Company | CompanyName, CompanySuffix, CatchPhrase, Bs |
| Lorem | Word, Sentence, Paragraph, Text, Lines |
| Date | Past, Future, Recent, Soon, Between |
| Random | Number, Bool, Guid, Hash, AlphaNumeric, String |
| System | FileName, FileExt, MimeType, Semver, Version |
