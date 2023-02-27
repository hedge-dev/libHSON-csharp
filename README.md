# libHSON-csharp

C# serialization library for the Hedgehog Set Object Notation ("HSON") format.

Currently, this library targets .NET Standard 2.1, which allows it to be used with all of the following .NET implementations (taken from [Microsoft's official documentation](https://learn.microsoft.com/en-us/dotnet/standard/net-standard?tabs=net-standard-2-1)):

| Implementation        | Version support   |
| --------------------- | ----------------- |
| .NET                  | 3.0+              |
| Mono                  | 6.4+              |
| Xamarin.iOS           | 12.16+            |
| Xamarin.Mac           | 5.16+             |
| Xamarin.Android       | 10.0+             |
| Unity                 | 2021.2+           |

**Note that the legacy .NET Framework is not supported.** This is due to us using features such as [nullable references](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references), which are not officially supported in .NET Framework.

## Examples

Read a HSON project from "input.hson", and print information on each of its objects to the console:

```csharp
using libHSON;
using System.Numerics;

// Read HSON project from input.hson
var project = Project.FromFile("input.hson");

// Print some information about all objects in the HSON project to the console
foreach (var obj in project.Objects)
{
    // Decompose the object's global transform into
    // globalScale, globalRotation, and globalPosition.
    Matrix4x4.Decompose(obj.GlobalTransform, out var globalScale,
        out var globalRotation, out var globalPosition);

    // Print information on this object to the console.
    Console.WriteLine("{");
    Console.WriteLine($"  id: {{{obj.Id}}}");
    Console.WriteLine($"  name: \"{obj.Name}\"");
    Console.WriteLine($"  parent: {{{obj.ParentId}}}");
    Console.WriteLine($"  instanceOf: {{{obj.InstanceOfId}}}");
    Console.WriteLine($"  type: \"{obj.Type}\"");
    Console.WriteLine($"  globalPosition: {globalPosition}");
    Console.WriteLine($"  globalRotation: {globalRotation}");
    Console.WriteLine($"  globalScale: {globalScale}");
    Console.WriteLine($"  localPosition: {obj.LocalPosition}");
    Console.WriteLine($"  localRotation: {obj.LocalRotation}");
    Console.WriteLine($"  localScale: {obj.LocalScale}");
    Console.WriteLine($"  isEditorVisible: {obj.IsEditorVisible}");
    Console.WriteLine($"  isExcluded: {obj.IsExcluded}");
    Console.WriteLine("}");
}
```

Detailed demonstration on how to parse a HSON project from a UTF-8 string and access its objects and parameters:

```csharp
using libHSON;

// Parse HSON project from a UTF-8 string containing HSON-formatted data.
// NOTE: UTF-8 string literals is a C# 11 feature. In older versions of
// C#, you could wrap the string in System.Text.Encoding.UTF8.GetBytes() 
var project = Project.FromData(@"
{
  ""objects"": [
    {
      ""id"": ""{AAAAAAAA-AAAA-AAAA-AAAA-AAAAAAAAAAAA}"",
      ""type"": ""DashPanel"",
      ""parameters"": {
        ""tags"": {
          ""RangeSpawning"": {
            ""rangeIn"": 140,
            ""rangeOut"": 20
          }
        },
        ""ocTime"": 0.25,
        ""speed"": 400,
        ""isVisible"": true
      }
    },
    {
      ""id"": ""{BBBBBBBB-BBBB-BBBB-BBBB-BBBBBBBBBBBB}"",
      ""instanceOf"": ""{AAAAAAAA-AAAA-AAAA-AAAA-AAAAAAAAAAAA}"",
      ""parameters"": {
        ""speed"": 200,
        ""isCyloopOn"": false
      }
    }
  ]
}"u8);

// Specific objects can be accessed by their GUID, or by their index
// This is possible since the order the objects were added in is preserved.
var obj1 = project.Objects[Guid.Parse("AAAAAAAA-AAAA-AAAA-AAAA-AAAAAAAAAAAA")];
var obj2 = project.Objects[Guid.Parse("BBBBBBBB-BBBB-BBBB-BBBB-BBBBBBBBBBBB")];

// To get the same objects by index, we'd do this instead:
// var obj1 = project.Objects[0];
// var obj2 = project.Objects[1];

// Parameters can be accessed via their names.
var ocTime = obj1.LocalParameters["ocTime"];

// You can query their type and get/set their values.
Console.WriteLine("some obj1 parameters:");
if (ocTime.Type == ParameterType.FloatingPoint)
{
    // Print the value of ocTime as a floating point.
    Console.WriteLine($"ocTime: {ocTime.ValueFloatingPoint}");

    // Change the value of ocTime from 0.25 to 0.5:
    ocTime.ValueFloatingPoint = 0.5;

    // We can also change the type of parameters as we wish,
    // by simply assigning a new value of a different type.

    // Change the value of ocTime again, this time from 0.5 to 7:
    // (will change .Type to ParameterType.SignedInteger)
    ocTime.ValueSignedInteger = 7;
}

// Print the values of the "speed" and "isVisible" parameters directly.
// You can use .Value to get the value as a System.Object, regardless of its actual type.
Console.WriteLine($"speed: {obj1.LocalParameters["speed"].Value}");
Console.WriteLine($"isVisible: {obj1.LocalParameters["isVisible"].Value}");

// Get the "tags" parameter, then its child "RangeSpawning",
// and finally, *its* child, "rangeIn".
var tags = obj1.LocalParameters["tags"];
var rangeSpawning = tags.ValueObject["RangeSpawning"];
var rangeIn = rangeSpawning.ValueObject["rangeIn"];

Console.WriteLine($"rangeIn: {rangeIn.ValueFloatingPoint}");

// You can also access nested parameters via a "path", like so:
Console.WriteLine($"rangeOut: {obj1.LocalParameters["tags/RangeSpawning/rangeOut"].Value}");

// The reason it's called "LocalParameters" is because it *only* contains
// the parameters that are *local* to this object; it does NOT contain parameters
// inherited from other objects!!!

// So this, for example, would fail with a KeyNotFoundException():
// ocTime = obj2.LocalParameters["ocTime"]; 

// (Because this parameter is not local to obj2; it is inherited from obj1)

// To get parameters with respect to inheritance, we can use the .GetParameter method, like so:
var isCyloopOn = obj2.GetParameter("isCyloopOn");   // local parameter
var ocTime2 = obj2.GetParameter("ocTime");          // inherited parameter (from obj1)

// It also supports paths, as you would expect:
var rangeIn2 = obj2.GetParameter("tags/RangeSpawning/rangeIn"); // inherited parameter (from obj1)

// If you want a collection of *all* parameters (both local and inherited parameters),
// you can use the .GetFlattenedParameters method, like so:
Console.WriteLine();
Console.WriteLine("some obj2 parameters:");

foreach (var param in obj2.GetFlattenedParameters())
{
    Console.WriteLine($"{param.Key}: {param.Value}");
}

// If you're wondering why .GetFlattenedParameters is a method (and not a property),
// it's because you can optionally pass in a "rootPath" parameter to it, like so:
var rangeSpawningParams = obj2.GetFlattenedParameters("tags/RangeSpawning");

// This variable will contain only the following parameters (both inherited from obj1, in this case):
// "rangeIn"
// "rangeOut"
```

Create a HSON project, add metadata and three objects to it, and save it to "output.hson":

```csharp
using libHSON;
using System.Numerics;
using System.Text.Json;

// Setup project
var project = new Project
{
    Metadata = new ProjectMetadata
    {
        Name = "libHSON Test",
        Author = "Radfordhound",
        Date = DateTime.UtcNow,
        Version = "1.0.0",
        Description = "Simple test layout intended for CyberSpace 1-1 from Sonic Frontiers."
    }
};

// Setup object "StartPosition1"
var startPosition1 = new libHSON.Object(
    type: "StartPosition",
    name: "StartPosition1",
    position: new Vector3(-600.0f, 240.5f, -400.0f)
);

startPosition1.LocalParameters.Add("tags/RangeSpawning/rangeIn", new Parameter(10000.0f));
startPosition1.LocalParameters.Add("tags/RangeSpawning/rangeOut", new Parameter(100.0f));

startPosition1.LocalParameters.Add("m_startType", new Parameter("FALL"));

project.Objects.Add(startPosition1);

// Setup object "DashPanel1"
var dashPanel1 = new libHSON.Object(
    type: "DashPanel",
    name: "DashPanel1",
    position: new Vector3(-582.1400146484375f, 135.52000427246094f, -265.260986328125f),
    rotation: new Quaternion(0.0f, 0.258819043636322f, 0.0f, 0.9659258127212524f)
);

dashPanel1.LocalParameters.Add("tags/RangeSpawning/rangeIn", new Parameter(140.0f));
dashPanel1.LocalParameters.Add("tags/RangeSpawning/rangeOut", new Parameter(20.0f));

dashPanel1.LocalParameters.Add("ocTime", new Parameter(0.25f));
dashPanel1.LocalParameters.Add("speed", new Parameter(400.0f));
dashPanel1.LocalParameters.Add("isVisible", new Parameter(true));

project.Objects.Add(dashPanel1);

// Setup object "DashPanel2"
var dashPanel2 = new libHSON.Object(
    type: "DashPanel",
    name: "DashPanel2",
    parent: dashPanel1, // This object's position/rotation/scale is local to DashPanel1
    instanceOf: dashPanel1, // This object inherits all of its non-specified parameters from DashPanel1
    position: new Vector3(-2.0f, 0.0f, 0.0f),
    rotation: libHSON.Object.DefaultRotation
);

project.Objects.Add(dashPanel2);

// Save HSON to output.hson, with JSON pretty-printing
project.Save("output.hson",
    jsonOptions: new JsonWriterOptions
    {
        Indented = true,
    });
```
