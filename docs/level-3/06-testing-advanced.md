# 06 · Testing Advanced (Moq, integration tests)

Level 1/2 covered basic unit tests with xUnit. This module adds mocking with
Moq, `Theory`/`InlineData` for parameterized tests, and integration tests
against a real ASP.NET Core pipeline with `WebApplicationFactory`.

## Project setup

```bash
dotnet new xunit -o OrderService.Tests
cd OrderService.Tests
dotnet add package Moq
dotnet add reference ../OrderService/OrderService.csproj
```

## Mocking dependencies with Moq

```csharp
public interface IOrderRepository
{
    void Save(Order order);
    Order? Find(Guid id);
}

public record Order(Guid Id, string Customer, decimal Total);

public class OrderService
{
    private readonly IOrderRepository _repository;
    public OrderService(IOrderRepository repository) => _repository = repository;

    public Order Place(string customer, decimal total)
    {
        if (total <= 0) throw new ArgumentException("Total must be positive.", nameof(total));
        var order = new Order(Guid.NewGuid(), customer, total);
        _repository.Save(order);
        return order;
    }
}
```

```csharp
using Moq;
using Xunit;

public class OrderServiceTests
{
    [Fact]
    public void Place_SavesOrderToRepository()
    {
        var repositoryMock = new Mock<IOrderRepository>();
        var service = new OrderService(repositoryMock.Object);

        var order = service.Place("Alice", 49.99m);

        repositoryMock.Verify(r => r.Save(It.Is<Order>(o => o.Customer == "Alice" && o.Total == 49.99m)), Times.Once);
        Assert.Equal("Alice", order.Customer);
    }

    [Fact]
    public void Place_ZeroTotal_Throws()
    {
        var repositoryMock = new Mock<IOrderRepository>();
        var service = new OrderService(repositoryMock.Object);

        Assert.Throws<ArgumentException>(() => service.Place("Bob", 0m));
        repositoryMock.Verify(r => r.Save(It.IsAny<Order>()), Times.Never);
    }
}
```

`Mock<IOrderRepository>()` creates a fake implementation with no real
storage. `.Object` hands the fake to the code under test. `Verify(...,
Times.Once)` asserts the mocked method was called exactly once with an
argument matching the `It.Is<Order>(...)` predicate — this checks
*behavior* (was `Save` called correctly) rather than state.

## Stubbing return values with `Setup`

```csharp
[Fact]
public void Find_ReturnsRepositoryResult()
{
    var existing = new Order(Guid.NewGuid(), "Carol", 10m);
    var repositoryMock = new Mock<IOrderRepository>();
    repositoryMock.Setup(r => r.Find(existing.Id)).Returns(existing);

    var found = repositoryMock.Object.Find(existing.Id);

    Assert.Equal(existing, found);
}
```

`Setup(...).Returns(...)` tells the mock what to return for a given call
shape; any call not matching a `Setup` returns the type's default (`null`
here) instead of throwing, unless `MockBehavior.Strict` is used.

## `Theory` and `InlineData` for parameterized tests

```csharp
public class DiscountCalculatorTests
{
    [Theory]
    [InlineData(100, 0, 100)]
    [InlineData(100, 10, 90)]
    [InlineData(50, 50, 25)]
    public void ApplyPercentage_ComputesExpectedTotal(decimal subtotal, decimal percent, decimal expected)
    {
        decimal result = subtotal * (1 - percent / 100m);
        Assert.Equal(expected, result);
    }

    [Theory]
    [MemberData(nameof(InvalidPercentages))]
    public void ApplyPercentage_RejectsOutOfRange(decimal percent)
    {
        Assert.Throws<ArgumentOutOfRangeException>(() =>
        {
            if (percent is < 0 or > 100) throw new ArgumentOutOfRangeException(nameof(percent));
        });
    }

    public static IEnumerable<object[]> InvalidPercentages =>
        new List<object[]> { new object[] { -1m }, new object[] { 101m } };
}
```

One `[Theory]` method with several `[InlineData]` rows runs as several
distinct test cases in the test explorer/CLI output — far less duplication
than copy-pasting a `[Fact]` per case. `[MemberData]` pulls cases from a
static property when they're too complex for inline literals.

## Integration testing an ASP.NET Core app

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

```csharp
// In the API project's Program.cs, add at the very end so the test project can see it:
// public partial class Program { }
```

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net;
using Xunit;

public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    public ProductsApiTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task GetProducts_ReturnsOk()
    {
        var client = _factory.CreateClient();

        var response = await client.GetAsync("/products");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task GetProduct_UnknownId_ReturnsNotFound()
    {
        var client = _factory.CreateClient();

        var response = await client.GetAsync("/products/999999");

        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

`WebApplicationFactory<Program>` spins up the whole app — routing,
middleware, DI container — in memory, backed by an in-process `TestServer`.
`IClassFixture<T>` shares one factory instance across all tests in the class
(the app starts once, not per test), which keeps the suite fast.

## Swapping real services for test doubles in integration tests

```csharp
public class ProductsApiWithFakeDbTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ProductsApiWithFakeDbTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.RemoveAll<IProductRepository>();
                services.AddSingleton<IProductRepository, InMemoryProductRepository>();
            });
        });
    }

    [Fact]
    public async Task GetProducts_UsesFakeRepository()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/products");
        response.EnsureSuccessStatusCode();
    }
}
```

`WithWebHostBuilder` lets a test replace real registrations (a SQL-backed
repository, an external email sender) with in-memory fakes before the app
boots, so integration tests stay hermetic — no real database or network
calls needed.

## Exercise

Given an `IPaymentGateway` interface with `Task<bool> ChargeAsync(decimal
amount)`, write unit tests for a `CheckoutService` that: (1) mocks a
successful charge and asserts an order is marked `Paid`; (2) mocks a failed
charge (`Returns(false)` — or `ThrowsAsync` for a gateway exception) and
asserts the order is marked `Failed` and no confirmation email is sent
(verify a mocked `IEmailSender.Send` was `Times.Never` called). Then add one
`WebApplicationFactory` integration test hitting a `POST /checkout` endpoint
with the real gateway swapped for a fake that always succeeds.
