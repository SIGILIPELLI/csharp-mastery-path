# 06 · Testing with xUnit

Automated tests catch regressions before your users do. xUnit is the most
common test framework in the modern .NET ecosystem, and it's what the
`dotnet new xunit` template scaffolds for you.

## Setting up a test project

```bash
dotnet new xunit -o Calculator.Tests
cd Calculator.Tests
dotnet add reference ../Calculator/Calculator.csproj
dotnet test
```

This creates a project referencing `xunit`, `xunit.runner.visualstudio`, and
`Microsoft.NET.Test.Sdk` — everything `dotnet test` needs to discover and run
tests.

## The code under test

```csharp
public static class Calculator
{
    public static int Add(int a, int b) => a + b;

    public static int Divide(int a, int b)
    {
        if (b == 0) throw new DivideByZeroException("Cannot divide by zero.");
        return a / b;
    }

    public static bool IsPrime(int n)
    {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++)
            if (n % i == 0) return false;
        return true;
    }
}
```

## A first test class

xUnit tests are plain methods marked `[Fact]` — no base class required:

```csharp
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_ReturnsSum()
    {
        int result = Calculator.Add(2, 3);
        Assert.Equal(5, result);
    }

    [Fact]
    public void Divide_ByZero_Throws()
    {
        Assert.Throws<DivideByZeroException>(() => Calculator.Divide(10, 0));
    }

    [Fact]
    public void Divide_ValidInputs_ReturnsQuotient()
    {
        Assert.Equal(4, Calculator.Divide(20, 5));
    }
}
```

`Assert.Equal(expected, actual)` — note the order: expected first. Getting it
backwards still passes or fails correctly, but produces a confusing failure
message ("Expected: 4, Actual: 5" reads oddly if reversed).

## Data-driven tests with `[Theory]`

Instead of writing near-identical `[Fact]` methods for each input, `[Theory]`
plus `[InlineData]` runs the same test body against a table of cases:

```csharp
public class PrimeTests
{
    [Theory]
    [InlineData(2, true)]
    [InlineData(4, false)]
    [InlineData(17, true)]
    [InlineData(1, false)]
    [InlineData(0, false)]
    [InlineData(-5, false)]
    public void IsPrime_ReturnsExpected(int input, bool expected)
    {
        Assert.Equal(expected, Calculator.IsPrime(input));
    }
}
```

`dotnet test` runs this as six independent test results, each reported
separately (`IsPrime_ReturnsExpected(input: 2, expected: True)` and so on) —
a failure in one case doesn't hide the others.

## Common assertions

```csharp
Assert.Equal(5, 2 + 3);
Assert.NotEqual(4, 2 + 3);
Assert.True(Calculator.IsPrime(7));
Assert.False(Calculator.IsPrime(8));
Assert.Null((string?)null);
Assert.NotNull("hello");
Assert.Contains(3, new[] { 1, 2, 3 });
Assert.Empty(new List<int>());
Assert.IsType<int>(42);
```

## Arrange, Act, Assert

Structuring each test in three clear parts keeps them readable as they grow:

```csharp
public class OrderTests
{
    [Fact]
    public void ApplyDiscount_ReducesTotalByPercentage()
    {
        // Arrange
        var order = new Order();
        order.AddItem("Widget", price: 100m, quantity: 2);

        // Act
        order.ApplyDiscount(percent: 10);

        // Assert
        Assert.Equal(180m, order.Total);
    }
}

public class Order
{
    private readonly List<(string Name, decimal Price, int Quantity)> _items = new();
    public decimal Total { get; private set; }

    public void AddItem(string name, decimal price, int quantity)
    {
        _items.Add((name, price, quantity));
        Total += price * quantity;
    }

    public void ApplyDiscount(decimal percent)
    {
        Total -= Total * percent / 100m;
    }
}
```

## Setup shared across tests: constructors and `IDisposable`

Each `[Fact]`/`[Theory]` method runs against a *new instance* of the test
class, so putting setup in the constructor gives every test a clean slate
without a separate `[SetUp]` attribute:

```csharp
public class OrderWithSetupTests : IDisposable
{
    private readonly Order _order;

    public OrderWithSetupTests()
    {
        // runs before every test in this class
        _order = new Order();
        _order.AddItem("Base item", price: 50m, quantity: 1);
    }

    [Fact]
    public void StartsWithBaseItemTotal()
    {
        Assert.Equal(50m, _order.Total);
    }

    [Fact]
    public void AddingSecondItemIncreasesTotal()
    {
        _order.AddItem("Extra", price: 20m, quantity: 1);
        Assert.Equal(70m, _order.Total);
    }

    public void Dispose()
    {
        // runs after every test — release files, connections, etc.
    }
}
```

## Exercise

Write `Calculator.Tests` for a `StringUtils.Reverse(string)` and
`StringUtils.IsPalindrome(string)` you implement yourself. Cover `Reverse`
with at least one `[Fact]`, and cover `IsPalindrome` with a `[Theory]` /
`[InlineData]` table including an empty string, a single character, a mixed
case word like `"Level"`, and a phrase with spaces. Run `dotnet test` and
confirm every case passes.
