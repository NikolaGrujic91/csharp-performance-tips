# C# Performance tips

List of quick performance improvements in C# collected from various online sources. Remember to always measure the performance of your code.

## Provide list capacity when possible

It is recommended to provide a capacity when creating a List or a collection instance. The .NET implementation of collections usually stores the values in an array that needs to be resized when the new elements are added. It means following:

1. A new array is allocated
2. The former values are copied to the new array
3. The former array is no more referenced, it is cleaned by Garbage Collector

**Allocation is cheap but Garbage Collection is expensive.**
More allocations mean more garbage collection and garbage collection introduce pauses.

```csharp
int capacity = 1000;
var newList = new List<...>(capacity);
```

## Prefer StringBuilder for string concatenation

Since the string class is immutable, each time concatenation is applied, the .NET framework ends up creating a new string.

For string concatenation, avoid using **Concat**, **+** or **+=**. This is especially important in loops or methods that are called very often.

Prefer to use **StringBuilder** and pre-allocate capacity if possible.

```csharp
int capacity = 1000;
var stringBuilder = new StringBuilder(capacity);

for (int i = 0; i < size; i++)
{
    stringBuilder.Append("example text");
}
```

## Prefer string.Compare for string comparison

Calling ToUpper() or ToLower() creates a temporary string. Avoid calling them just for string comparison such as:

```csharp
if (transactionID.ToLowerInvariant() == "undefined")
```

Instead use string.Compare:

```csharp
if (string.Compare(transactionID, "undefined", StringComparison.OrdinalIgnoreCase) == 0)
```

## Avoid unnecessary boxing and unboxing

Boxing and unboxing are, like garbage collection, expensive processes. Boxing consists of converting a value type to **object** or to an interface type this value type implements. Unboxing is the opposite, it extracts the value type from object.

```csharp
int BoxUnboxValueType()
{
   int i = 10;
   object o = (object)i; //i is Boxed
   return (int)o + 3; //i is Unboxed
}
```

When a value is boxed another object is created on the heap, which puts additional pressure on the Garbage Collector.

One way to avoid is to prefer using generic collections such as:

```csharp
System.Collections.Generic.List<T>
```

instead of

```csharp
System.Collections.ArrayList
```

## Avoid empty destructors

Do not add empty destructors to the classes. An entry is added to the **Finalize** queue for every class that has a destructor. Garbage Collector is called to process the queue when the destructor is called. An empty destructor means this is all for nothing. Do not create work for the Garbage Collector unnecessarily.

## Closures, Lambdas and LINQ

When using closures compiler rewrites to capture local variables as a new class. Lambda is rewritten as a method on this class. All of this means there can be a lot of heap allocations for each capture.

Closures - Avoid in critical paths. Pass state as arguments to lambda.

LINQ - Avoid in critical paths. Use ```foreach``` and ```if``` instead.

## Range check elimination

One of the many benefits of managed code is automatic range checking; every time you access an array using array[index] semantics, the JIT emits a check to make sure that the index is in the bounds of the array. In the context of loops with a large number of iterations and small number of instructions executed per iteration these range checks can be expensive. There are cases when the JIT will detect that these range checks are unnecessary and will eliminate the check from the body of the loop, only checking it once before the loop execution begins. In C# there is a programmatic pattern to ensure that these range checks will be eliminated: **explicitly test for the length of the array in the "for" statement**. Note that subtle deviations from this pattern will result in the check not being eliminated, and in this case, adding a value to the index.

```csharp
//Range check will be eliminated
for(int i = 0; i < myArray.Length; i++) 
{
   Console.WriteLine(myArray[i].ToString());
}

//Range check will NOT be eliminated
for(int i = 0; i < myArray.Length + y; i++) 
{ 
   Console.WriteLine(myArray[i+x].ToString());
}
```

## Use AddRange to add groups

Use **AddRange** to add a whole collection, rather than adding each item in the collection iteratively. Nearly all windows controls and collections have both Add and AddRange methods, and each is optimized for a different purpose. Add is useful for adding a single item, whereas AddRange has some extra overhead but wins out when adding multiple items.


# High performance patterns

[https://prodotnetmemory.com/slides/performancepatternslong](https://prodotnetmemory.com/slides/performancepatternslong/#1)

## Frugal object

Represents zero, one or many strings in an efficient way. Saves memory footprint and traffic for lists with single element: doesn't allocate real list until number of elements is more then 1.

```csharp
public struct CompactList<T> : IEnumerable<T>
{
   private T singleValue;
   private List<T> multipleValues;
   ...
}

```

## Struct of Arrays

Processing a lot of data is not efficient due to the memory access time. Design data structures and processing steps to leverage data locality and sequential access. Most typically, in the form of plain arrays of data.

Array of Structs

```csharp
class CustomerClassRepository
{
  List<Customer> customers = new List<Customer>();
  public void UpdateScorings()
  {
    foreach (var customer in customers)
      customer.UpdateScoring();
  }
}
public class CustomerClass
{
  private double Earnings;
  private DateTime DateOfBirth;
  private bool IsSmoking;
  private double Scoring;
  private HealthData Health;
  private AuxiliaryData Auxiliary;
  private Company Employer;
  public void UpdateScoring()
  {
    Scoring = Earnings * (IsSmoking ? 0.8 : 1.0) * ProcessAge(DateOfBirth);
  }
  private double ProcessAge(DateTime dateOfBirth) => ...;
}
```

Instead use Struct of Arrays

```csharp
class CustomerRepositoryDOD
{
  int NumberOfCustomers;
  double[] Scoring;
  double[] Earnings;
  bool[] IsSmoking;
  int[] YearOfBirth;
  DateTime[] DateOfBirth;
  public void UpdateScorings()
  {
     for (int i = 0; i < NumberOfCustomers; ++i)
        Scoring[i] = Earnings[i] * (IsSmoking[i] ? 0.8 : 1.0) 
                     * ProcessAge(YearOfBirth[i]);
  }
  public double ProcessAge(int yearOfBirth) => ...;
}
```

## Fit the cache line

```csharp
class Program
{
  // NuGet - ObjectLayoutInspector
  static void Main(string[] args)
  {
     TypeLayout layout = ObjectLayoutInspector.TypeLayout.GetLayout<CustomerValue>();
     System.Diagnostics.Debug.WriteLine(layout.ToString(true));
  }

  [StructLayout(LayoutKind.Sequential)]
  public struct CustomerValue
  {
    double Earnings;
    double Scoring;
    int YearOfBirth;
    bool IsSmoking;
    int HealthDataId;
    int AuxiliaryDataId;
    int EmployerId;
  }
}
```