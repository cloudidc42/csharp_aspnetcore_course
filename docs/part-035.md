# Part 035: Reflection

## เนื้อหาใน Part นี้
- Type.GetType()
- GetProperties, GetMethods, GetFields
- Invoke methods dynamically
- Create instances dynamically
- Use cases: Plugin systems, ORM
- Performance considerations
- โปรแกรมตัวอย่าง: Simple Object Mapper

---

## 1. Type.GetType()

Reflection ช่วยให้เราสามารถตรวจสอบและใช้งาน types ในขณะ runtime

```csharp
using System;
using System.Collections.Generic;
using System.Reflection;

class TypeGetTypeExample
{
    static void Main()
    {
        // 3 วิธีหลักในการได้ Type object
        
        // 1. typeof operator (compile-time)
        Type type1 = typeof(string);
        Type type2 = typeof(List<int>);
        
        // 2. GetType() บน instance
        string text = "hello";
        Type type3 = text.GetType();
        
        var list = new List<int> { 1, 2, 3 };
        Type type4 = list.GetType();
        
        // 3. Type.GetType() จาก string
        Type? type5 = Type.GetType("System.String");
        Type? type6 = Type.GetType("System.Collections.Generic.List`1");
        
        // จาก assembly อื่น
        Type? type7 = Type.GetType("MyNamespace.MyClass, MyAssembly");
        
        Console.WriteLine($"typeof(string): {type1.FullName}");
        Console.WriteLine($"GetType() on string: {type3.FullName}");
        Console.WriteLine($"Same type: {type1 == type3}"); // true
        
        // Type properties
        Type intType = typeof(int);
        Console.WriteLine($"\nType info for int:");
        Console.WriteLine($"  Name: {intType.Name}");
        Console.WriteLine($"  FullName: {intType.FullName}");
        Console.WriteLine($"  Namespace: {intType.Namespace}");
        Console.WriteLine($"  Assembly: {intType.Assembly.GetName().Name}");
        Console.WriteLine($"  IsValueType: {intType.IsValueType}");
        Console.WriteLine($"  IsClass: {intType.IsClass}");
        Console.WriteLine($"  IsPrimitive: {intType.IsPrimitive}");
        Console.WriteLine($"  IsAbstract: {intType.IsAbstract}");
        Console.WriteLine($"  IsSealed: {intType.IsSealed}");
        Console.WriteLine($"  IsGenericType: {typeof(List<int>).IsGenericType}");
        
        // Type hierarchy
        Type objType = typeof(object);
        Type stringType = typeof(string);
        
        Console.WriteLine($"\nstring inherits from object: {stringType.IsSubclassOf(objType)}");
        Console.WriteLine($"string implements IComparable: {typeof(IComparable).IsAssignableFrom(stringType)}");
        
        // Generic types
        Type genericList = typeof(List<>);
        Type closedList = typeof(List<string>);
        
        Console.WriteLine($"\nList<> IsGenericTypeDefinition: {genericList.IsGenericTypeDefinition}");
        Console.WriteLine($"List<string> GenericArguments: {closedList.GenericTypeArguments[0].Name}");
        
        // Nullable types
        Type? nullableInt = typeof(int?);
        Console.WriteLine($"\nint? IsGenericType: {nullableInt.IsGenericType}");
        Console.WriteLine($"Underlying type: {Nullable.GetUnderlyingType(nullableInt)?.Name}");
    }
}
```

---

## 2. GetProperties, GetMethods, GetFields

```csharp
using System;
using System.Reflection;
using System.Linq;

public class SampleClass
{
    // Fields
    public string PublicField = "public";
    private int _privateField = 42;
    protected bool ProtectedField = true;
    public static string StaticField = "static";
    
    // Properties
    public string Name { get; set; } = "Default";
    public int Age { get; private set; }
    private string Secret { get; set; } = "secret";
    public static int Count { get; set; }
    
    // Methods
    public string GetInfo() => $"{Name}, {Age}";
    private void DoSomethingPrivate() { }
    public static void StaticMethod() { }
    public virtual string VirtualMethod() => "base";
    
    // Constructor
    public SampleClass() { }
    public SampleClass(string name, int age)
    {
        Name = name;
        Age = age;
    }
    
    // Events
    public event EventHandler? DataChanged;
}

class ReflectionInspector
{
    static void Main()
    {
        var type = typeof(SampleClass);
        
        InspectProperties(type);
        InspectMethods(type);
        InspectFields(type);
        InspectConstructors(type);
    }
    
    static void InspectProperties(Type type)
    {
        Console.WriteLine("=== Properties ===");
        
        // Public properties เท่านั้น
        var publicProps = type.GetProperties();
        
        // ทุก properties (รวม private)
        var allProps = type.GetProperties(
            BindingFlags.Public | BindingFlags.NonPublic |
            BindingFlags.Instance | BindingFlags.Static
        );
        
        foreach (var prop in allProps)
        {
            var getter = prop.GetGetMethod(nonPublic: true);
            var setter = prop.GetSetMethod(nonPublic: true);
            
            string access = getter?.IsPublic == true ? "public" :
                           getter?.IsFamily == true ? "protected" : "private";
            string isStatic = getter?.IsStatic == true ? " static" : "";
            string hasGet = getter != null ? "get; " : "";
            string hasSet = setter != null ? "set; " : "";
            
            Console.WriteLine($"  {access}{isStatic} {prop.PropertyType.Name} {prop.Name} " +
                $"{{ {hasGet}{hasSet}}}");
        }
    }
    
    static void InspectMethods(Type type)
    {
        Console.WriteLine("\n=== Methods ===");
        
        var methods = type.GetMethods(
            BindingFlags.Public | BindingFlags.NonPublic |
            BindingFlags.Instance | BindingFlags.Static |
            BindingFlags.DeclaredOnly // ไม่รวม inherited methods
        );
        
        foreach (var method in methods.Where(m => !m.IsSpecialName)) // ไม่รวม get_/set_ auto-generated
        {
            string access = method.IsPublic ? "public" :
                           method.IsFamily ? "protected" : "private";
            string isStatic = method.IsStatic ? " static" : "";
            string isVirtual = method.IsVirtual ? " virtual" : "";
            string isAbstract = method.IsAbstract ? " abstract" : "";
            
            var parameters = method.GetParameters()
                .Select(p => $"{p.ParameterType.Name} {p.Name}");
            
            Console.WriteLine($"  {access}{isStatic}{isVirtual}{isAbstract} " +
                $"{method.ReturnType.Name} {method.Name}({string.Join(", ", parameters)})");
        }
    }
    
    static void InspectFields(Type type)
    {
        Console.WriteLine("\n=== Fields ===");
        
        var fields = type.GetFields(
            BindingFlags.Public | BindingFlags.NonPublic |
            BindingFlags.Instance | BindingFlags.Static
        );
        
        foreach (var field in fields.Where(f => !f.Name.Contains(">"))) // ไม่รวม backing fields
        {
            string access = field.IsPublic ? "public" :
                           field.IsFamily ? "protected" : "private";
            string isStatic = field.IsStatic ? " static" : "";
            
            Console.WriteLine($"  {access}{isStatic} {field.FieldType.Name} {field.Name}");
        }
    }
    
    static void InspectConstructors(Type type)
    {
        Console.WriteLine("\n=== Constructors ===");
        
        foreach (var ctor in type.GetConstructors())
        {
            var parameters = ctor.GetParameters()
                .Select(p => $"{p.ParameterType.Name} {p.Name}");
            
            Console.WriteLine($"  {type.Name}({string.Join(", ", parameters)})");
        }
    }
}
```

---

## 3. Invoke Methods Dynamically

```csharp
using System;
using System.Reflection;

public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Multiply(double a, double b) => a * b;
    private string FormatResult(double value) => $"Result: {value:F2}";
    public static int StaticAdd(int a, int b) => a + b;
    public T Echo<T>(T value) => value;
}

class DynamicInvocation
{
    static void Main()
    {
        var calc = new Calculator();
        var type = typeof(Calculator);
        
        // เรียก public method
        var addMethod = type.GetMethod("Add");
        object? result = addMethod?.Invoke(calc, new object[] { 10, 20 });
        Console.WriteLine($"Add(10, 20) = {result}"); // 30
        
        // เรียก private method
        var formatMethod = type.GetMethod("FormatResult",
            BindingFlags.NonPublic | BindingFlags.Instance);
        object? formatted = formatMethod?.Invoke(calc, new object[] { 3.14159 });
        Console.WriteLine($"FormatResult = {formatted}");
        
        // เรียก static method
        var staticAdd = type.GetMethod("StaticAdd");
        object? staticResult = staticAdd?.Invoke(null, new object[] { 5, 7 });
        Console.WriteLine($"StaticAdd(5, 7) = {staticResult}");
        
        // เรียก generic method
        var echoMethod = type.GetMethod("Echo");
        var closedEcho = echoMethod?.MakeGenericMethod(typeof(string));
        object? echoResult = closedEcho?.Invoke(calc, new object[] { "Hello!" });
        Console.WriteLine($"Echo<string> = {echoResult}");
        
        // สร้าง delegate เพื่อ performance ดีกว่า Invoke
        var addDelegate = (Func<int, int, int>)Delegate.CreateDelegate(
            typeof(Func<int, int, int>),
            calc,
            addMethod!
        );
        
        int delegateResult = addDelegate(100, 200);
        Console.WriteLine($"Delegate Add = {delegateResult}");
    }
}
```

---

## 4. Create Instances Dynamically

```csharp
using System;
using System.Reflection;

public interface IPlugin
{
    string Name { get; }
    void Execute();
}

public class HelloPlugin : IPlugin
{
    public string Name => "Hello Plugin";
    public void Execute() => Console.WriteLine("Hello from Plugin!");
}

public class WorldPlugin : IPlugin
{
    private readonly string _message;
    
    public WorldPlugin(string message)
    {
        _message = message;
    }
    
    public string Name => "World Plugin";
    public void Execute() => Console.WriteLine($"World: {_message}");
}

class DynamicCreation
{
    static void Main()
    {
        var type = typeof(HelloPlugin);
        
        // 1. Activator.CreateInstance (ง่ายที่สุด)
        var instance1 = Activator.CreateInstance(type);
        
        // 2. Activator.CreateInstance กับ parameters
        var worldType = typeof(WorldPlugin);
        var instance2 = (IPlugin?)Activator.CreateInstance(worldType, "Dynamic Message");
        instance2?.Execute();
        
        // 3. ConstructorInfo.Invoke
        var ctor = worldType.GetConstructor(new[] { typeof(string) });
        var instance3 = (IPlugin?)ctor?.Invoke(new object[] { "Constructor Invoke" });
        instance3?.Execute();
        
        // 4. Generic version
        var instance4 = Activator.CreateInstance<HelloPlugin>();
        
        // 5. สร้าง generic type
        var listType = typeof(List<>).MakeGenericType(typeof(string));
        var dynamicList = Activator.CreateInstance(listType);
        
        // เรียก Add บน dynamic list
        var addMethod = listType.GetMethod("Add");
        addMethod?.Invoke(dynamicList, new object[] { "item1" });
        addMethod?.Invoke(dynamicList, new object[] { "item2" });
        
        var countProp = listType.GetProperty("Count");
        Console.WriteLine($"List count: {countProp?.GetValue(dynamicList)}");
        
        // Plugin factory pattern
        var plugins = new Dictionary<string, Type>
        {
            ["hello"] = typeof(HelloPlugin),
            ["world"] = typeof(WorldPlugin)
        };
        
        IPlugin? plugin = CreatePlugin(plugins, "world", "Custom Message");
        plugin?.Execute();
    }
    
    static IPlugin? CreatePlugin(Dictionary<string, Type> registry, string name, params object[] args)
    {
        if (!registry.TryGetValue(name, out Type? type))
            return null;
        
        return (IPlugin?)Activator.CreateInstance(type, args);
    }
}
```

---

## 5. Use Cases: Plugin System และ ORM

### Plugin System

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Reflection;

// Plugin Interface
public interface IPlugin
{
    string Name { get; }
    string Version { get; }
    void Initialize(IPluginContext context);
    void Execute(string command, Dictionary<string, object> parameters);
}

public interface IPluginContext
{
    void Log(string message);
    T GetService<T>() where T : class;
}

// Plugin Attribute
[AttributeUsage(AttributeTargets.Class)]
public class PluginMetadataAttribute : Attribute
{
    public string Name { get; }
    public string Description { get; }
    public string Author { get; }
    
    public PluginMetadataAttribute(string name, string description, string author)
    {
        Name = name;
        Description = description;
        Author = author;
    }
}

// Sample Plugins
[PluginMetadata("Logger", "Logs messages to console", "DevTeam")]
public class LoggerPlugin : IPlugin
{
    private IPluginContext? _context;
    
    public string Name => "Logger";
    public string Version => "1.0.0";
    
    public void Initialize(IPluginContext context) => _context = context;
    
    public void Execute(string command, Dictionary<string, object> parameters)
    {
        if (command == "log" && parameters.TryGetValue("message", out var msg))
        {
            _context?.Log($"[Logger Plugin] {msg}");
        }
    }
}

[PluginMetadata("Calculator", "Performs calculations", "DevTeam")]
public class CalculatorPlugin : IPlugin
{
    public string Name => "Calculator";
    public string Version => "2.0.0";
    
    public void Initialize(IPluginContext context) { }
    
    public void Execute(string command, Dictionary<string, object> parameters)
    {
        if (command == "add")
        {
            int a = Convert.ToInt32(parameters["a"]);
            int b = Convert.ToInt32(parameters["b"]);
            Console.WriteLine($"Calculator: {a} + {b} = {a + b}");
        }
    }
}

// Plugin Manager
public class PluginManager
{
    private readonly Dictionary<string, IPlugin> _plugins = new();
    private readonly IPluginContext _context;
    
    public PluginManager(IPluginContext context)
    {
        _context = context;
    }
    
    // Load plugins จาก assembly ปัจจุบัน
    public void LoadFromCurrentAssembly()
    {
        LoadFromAssembly(Assembly.GetExecutingAssembly());
    }
    
    // Load plugins จาก assembly file
    public void LoadFromFile(string assemblyPath)
    {
        var assembly = Assembly.LoadFrom(assemblyPath);
        LoadFromAssembly(assembly);
    }
    
    private void LoadFromAssembly(Assembly assembly)
    {
        var pluginTypes = assembly.GetTypes()
            .Where(t => t.IsClass && !t.IsAbstract)
            .Where(t => typeof(IPlugin).IsAssignableFrom(t))
            .ToList();
        
        foreach (var type in pluginTypes)
        {
            try
            {
                var plugin = (IPlugin?)Activator.CreateInstance(type);
                if (plugin == null) continue;
                
                plugin.Initialize(_context);
                _plugins[plugin.Name] = plugin;
                
                // อ่าน metadata
                var metadata = type.GetCustomAttribute<PluginMetadataAttribute>();
                Console.WriteLine($"Loaded: {plugin.Name} v{plugin.Version}" +
                    (metadata != null ? $" by {metadata.Author}" : ""));
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Failed to load {type.Name}: {ex.Message}");
            }
        }
    }
    
    public void Execute(string pluginName, string command, Dictionary<string, object> parameters)
    {
        if (_plugins.TryGetValue(pluginName, out var plugin))
        {
            plugin.Execute(command, parameters);
        }
        else
        {
            Console.WriteLine($"Plugin '{pluginName}' not found");
        }
    }
    
    public IEnumerable<(string Name, string Version, PluginMetadataAttribute? Metadata)> GetPluginInfo()
    {
        return _plugins.Values.Select(p => (
            p.Name,
            p.Version,
            p.GetType().GetCustomAttribute<PluginMetadataAttribute>()
        ));
    }
}
```

### Simple ORM

```csharp
using System;
using System.Collections.Generic;
using System.Reflection;
using System.Text;

// ORM Attributes (จาก Part 034)
[AttributeUsage(AttributeTargets.Class)]
public class TableAttribute : Attribute
{
    public string TableName { get; }
    public string? Schema { get; set; }
    public TableAttribute(string tableName) { TableName = tableName; }
}

[AttributeUsage(AttributeTargets.Property)]
public class ColumnAttribute : Attribute
{
    public string? ColumnName { get; set; }
    public bool IsPrimaryKey { get; set; }
    public bool IsNullable { get; set; } = true;
    public bool IsIdentity { get; set; }
    public int MaxLength { get; set; } = -1;
}

// ORM Implementation
public class SimpleOrm
{
    // Generate SELECT SQL
    public static string GenerateSelect<T>() where T : class
    {
        var type = typeof(T);
        var tableAttr = type.GetCustomAttribute<TableAttribute>();
        if (tableAttr == null) throw new InvalidOperationException($"{type.Name} ไม่มี [Table] attribute");
        
        var tableName = tableAttr.Schema != null
            ? $"[{tableAttr.Schema}].[{tableAttr.TableName}]"
            : $"[{tableAttr.TableName}]";
        
        var columns = GetMappedColumns(type);
        var columnList = string.Join(", ", columns.Select(c =>
        {
            var colName = c.attr.ColumnName ?? c.prop.Name.ToLower();
            return c.attr.ColumnName != null
                ? $"[{c.attr.ColumnName}] AS [{c.prop.Name}]"
                : $"[{colName}]";
        }));
        
        return $"SELECT {columnList} FROM {tableName}";
    }
    
    // Generate INSERT SQL
    public static (string Sql, Dictionary<string, object?> Parameters) GenerateInsert<T>(T entity)
        where T : class
    {
        var type = typeof(T);
        var tableAttr = type.GetCustomAttribute<TableAttribute>()
            ?? throw new InvalidOperationException($"{type.Name} ไม่มี [Table] attribute");
        
        var tableName = $"[{tableAttr.TableName}]";
        var columns = GetMappedColumns(type)
            .Where(c => !c.attr.IsPrimaryKey || !c.attr.IsIdentity)
            .ToList();
        
        var colNames = columns.Select(c => $"[{c.attr.ColumnName ?? c.prop.Name.ToLower()}]");
        var paramNames = columns.Select(c => $"@{c.prop.Name}");
        var parameters = columns.ToDictionary(
            c => c.prop.Name,
            c => c.prop.GetValue(entity)
        );
        
        var sql = $"INSERT INTO {tableName} ({string.Join(", ", colNames)}) " +
                  $"VALUES ({string.Join(", ", paramNames)})";
        
        return (sql, parameters);
    }
    
    // Map DataReader to objects
    public static T MapFromReader<T>(System.Data.IDataReader reader) where T : new()
    {
        var obj = new T();
        var type = typeof(T);
        var columns = GetMappedColumns(type);
        
        for (int i = 0; i < reader.FieldCount; i++)
        {
            string fieldName = reader.GetName(i);
            
            // หา property ที่ตรงกับ column
            var mapping = columns.FirstOrDefault(c =>
                (c.attr.ColumnName ?? c.prop.Name.ToLower())
                    .Equals(fieldName, StringComparison.OrdinalIgnoreCase));
            
            if (mapping != default && !reader.IsDBNull(i))
            {
                var value = reader.GetValue(i);
                var propType = mapping.prop.PropertyType;
                
                // Handle nullable types
                if (propType.IsGenericType && propType.GetGenericTypeDefinition() == typeof(Nullable<>))
                    propType = Nullable.GetUnderlyingType(propType)!;
                
                mapping.prop.SetValue(obj, Convert.ChangeType(value, propType));
            }
        }
        
        return obj;
    }
    
    private static IEnumerable<(PropertyInfo prop, ColumnAttribute attr)> GetMappedColumns(Type type)
    {
        return type.GetProperties()
            .Select(p => (prop: p, attr: p.GetCustomAttribute<ColumnAttribute>()!))
            .Where(x => x.attr != null);
    }
}
```

---

## 6. Performance Considerations

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq.Expressions;
using System.Reflection;
using System.Reflection.Emit;

class PerformanceComparison
{
    static void Main()
    {
        const int iterations = 1_000_000;
        var target = new SampleClass { Name = "Test" };
        var property = typeof(SampleClass).GetProperty("Name")!;
        
        // 1. Direct access
        MeasureTime("Direct access", iterations, () =>
        {
            var _ = target.Name;
        });
        
        // 2. Reflection (ช้าที่สุด)
        MeasureTime("Reflection PropertyInfo.GetValue", iterations, () =>
        {
            var _ = property.GetValue(target);
        });
        
        // 3. Compiled Lambda (เร็วมาก - ใกล้เคียง direct)
        var getter = CreateGetter<SampleClass, string>(property);
        MeasureTime("Compiled Lambda", iterations, () =>
        {
            var _ = getter(target);
        });
        
        // 4. Cached Reflection
        var cachedGetter = property.GetGetMethod()!;
        MeasureTime("Cached MethodInfo.Invoke", iterations, () =>
        {
            var _ = cachedGetter.Invoke(target, null);
        });
    }
    
    // Compiled expression สำหรับ fast property access
    static Func<TSource, TResult> CreateGetter<TSource, TResult>(PropertyInfo property)
    {
        var parameter = Expression.Parameter(typeof(TSource), "obj");
        var propertyAccess = Expression.Property(parameter, property);
        var convert = Expression.Convert(propertyAccess, typeof(TResult));
        var lambda = Expression.Lambda<Func<TSource, TResult>>(convert, parameter);
        return lambda.Compile();
    }
    
    static void MeasureTime(string name, int iterations, Action action)
    {
        // Warmup
        for (int i = 0; i < 1000; i++) action();
        
        var sw = Stopwatch.StartNew();
        for (int i = 0; i < iterations; i++)
        {
            action();
        }
        sw.Stop();
        
        Console.WriteLine($"{name}: {sw.ElapsedMilliseconds}ms ({iterations:N0} iterations)");
    }
}

public class SampleClass
{
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
}

// Reflection Cache สำหรับ performance
public class ReflectionCache
{
    private static readonly Dictionary<Type, PropertyInfo[]> _propertyCache = new();
    private static readonly Dictionary<(Type, string), PropertyInfo?> _namedPropertyCache = new();
    private static readonly object _lock = new();
    
    public static PropertyInfo[] GetProperties(Type type)
    {
        if (!_propertyCache.TryGetValue(type, out var props))
        {
            lock (_lock)
            {
                if (!_propertyCache.TryGetValue(type, out props))
                {
                    props = type.GetProperties();
                    _propertyCache[type] = props;
                }
            }
        }
        return props;
    }
    
    public static PropertyInfo? GetProperty(Type type, string name)
    {
        var key = (type, name);
        if (!_namedPropertyCache.TryGetValue(key, out var prop))
        {
            lock (_lock)
            {
                if (!_namedPropertyCache.TryGetValue(key, out prop))
                {
                    prop = type.GetProperty(name);
                    _namedPropertyCache[key] = prop;
                }
            }
        }
        return prop;
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Simple Object Mapper

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;
using System.Reflection;

// Object Mapper ที่แมป properties ระหว่าง objects

[AttributeUsage(AttributeTargets.Property)]
public class MapFromAttribute : Attribute
{
    public string SourcePropertyName { get; }
    public MapFromAttribute(string sourcePropertyName)
    {
        SourcePropertyName = sourcePropertyName;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class IgnoreMapAttribute : Attribute { }

// Source/Destination Models
public class UserEntity
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string EmailAddress { get; set; } = string.Empty;
    public DateTime BirthDate { get; set; }
    public string PasswordHash { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class UserDto
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    
    [MapFrom("EmailAddress")]
    public string Email { get; set; } = string.Empty;
    
    [IgnoreMap]
    public string PasswordHash { get; set; } = string.Empty; // ไม่แมป
    
    public bool IsActive { get; set; }
    
    // Computed: ไม่มีใน source
    public string FullName => $"{FirstName} {LastName}";
    public int Age => (int)((DateTime.Today - BirthDate).TotalDays / 365.25);
    
    [MapFrom("BirthDate")]
    public DateTime BirthDate { get; set; }
}

public class UserSummary
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty; // จะ map จาก computed property
    public string Email { get; set; } = string.Empty;
    public bool IsActive { get; set; }
}

// Object Mapper
public class ObjectMapper
{
    // Cache mapping configurations
    private static readonly Dictionary<(Type, Type), MappingConfig> _configs = new();
    
    // Map single object
    public static TDest Map<TSource, TDest>(TSource source) where TDest : new()
    {
        var config = GetOrCreateConfig(typeof(TSource), typeof(TDest));
        var dest = new TDest();
        config.Apply(source!, dest);
        return dest;
    }
    
    // Map collection
    public static IEnumerable<TDest> MapList<TSource, TDest>(IEnumerable<TSource> source)
        where TDest : new()
    {
        return source.Select(Map<TSource, TDest>);
    }
    
    // Map into existing object
    public static void MapInto<TSource, TDest>(TSource source, TDest dest)
    {
        var config = GetOrCreateConfig(typeof(TSource), typeof(TDest));
        config.Apply(source!, dest!);
    }
    
    private static MappingConfig GetOrCreateConfig(Type sourceType, Type destType)
    {
        var key = (sourceType, destType);
        if (!_configs.TryGetValue(key, out var config))
        {
            config = new MappingConfig(sourceType, destType);
            _configs[key] = config;
        }
        return config;
    }
}

// Mapping Configuration - built at startup, reused for performance
public class MappingConfig
{
    private readonly List<Action<object, object>> _mappingActions = new();
    
    public MappingConfig(Type sourceType, Type destType)
    {
        BuildMappings(sourceType, destType);
    }
    
    private void BuildMappings(Type sourceType, Type destType)
    {
        var destProps = destType.GetProperties(BindingFlags.Public | BindingFlags.Instance)
            .Where(p => p.CanWrite)
            .Where(p => p.GetCustomAttribute<IgnoreMapAttribute>() == null);
        
        var sourceProps = sourceType.GetProperties(BindingFlags.Public | BindingFlags.Instance)
            .Where(p => p.CanRead)
            .ToDictionary(p => p.Name, p => p, StringComparer.OrdinalIgnoreCase);
        
        foreach (var destProp in destProps)
        {
            // ตรวจสอบ [MapFrom] attribute
            var mapFromAttr = destProp.GetCustomAttribute<MapFromAttribute>();
            string sourcePropName = mapFromAttr?.SourcePropertyName ?? destProp.Name;
            
            if (!sourceProps.TryGetValue(sourcePropName, out var sourceProp))
                continue;
            
            // สร้าง compiled mapper action
            var action = CreateMappingAction(sourceProp, destProp);
            if (action != null)
                _mappingActions.Add(action);
        }
    }
    
    private Action<object, object>? CreateMappingAction(PropertyInfo sourceProp, PropertyInfo destProp)
    {
        // Type compatibility check
        var sourceType = sourceProp.PropertyType;
        var destType = destProp.PropertyType;
        
        // Exact match หรือ assignable
        if (!destType.IsAssignableFrom(sourceType) &&
            !CanConvert(sourceType, destType))
            return null;
        
        // สร้าง compiled expression
        var sourceParam = Expression.Parameter(typeof(object), "source");
        var destParam = Expression.Parameter(typeof(object), "dest");
        
        var castSource = Expression.Convert(sourceParam, sourceProp.DeclaringType!);
        var castDest = Expression.Convert(destParam, destProp.DeclaringType!);
        
        var getValue = Expression.Property(castSource, sourceProp);
        
        Expression convertedValue;
        if (destType.IsAssignableFrom(sourceType))
        {
            convertedValue = sourceType == destType
                ? (Expression)getValue
                : Expression.Convert(getValue, destType);
        }
        else
        {
            // Use Convert.ChangeType for compatible types
            var changeType = typeof(Convert).GetMethod("ChangeType", new[] { typeof(object), typeof(Type) })!;
            convertedValue = Expression.Convert(
                Expression.Call(changeType,
                    Expression.Convert(getValue, typeof(object)),
                    Expression.Constant(destType)),
                destType
            );
        }
        
        var setValue = Expression.Call(castDest, destProp.GetSetMethod()!, convertedValue);
        var lambda = Expression.Lambda<Action<object, object>>(setValue, sourceParam, destParam);
        
        return lambda.Compile();
    }
    
    private static bool CanConvert(Type from, Type to)
    {
        var convertible = new[] { typeof(int), typeof(long), typeof(double), typeof(float),
                                   typeof(decimal), typeof(string), typeof(bool) };
        return Array.IndexOf(convertible, from) >= 0 && Array.IndexOf(convertible, to) >= 0;
    }
    
    public void Apply(object source, object dest)
    {
        foreach (var action in _mappingActions)
            action(source, dest);
    }
}

// Main Program
class Program
{
    static void Main()
    {
        Console.WriteLine("===== Object Mapper Demo =====\n");
        
        var entity = new UserEntity
        {
            Id = 1,
            FirstName = "สมชาย",
            LastName = "ใจดี",
            EmailAddress = "somchai@example.com",
            BirthDate = new DateTime(1990, 5, 15),
            PasswordHash = "hashed_password_here",
            IsActive = true,
            CreatedAt = DateTime.Now
        };
        
        // Map Entity -> DTO
        var dto = ObjectMapper.Map<UserEntity, UserDto>(entity);
        Console.WriteLine("Entity -> DTO:");
        Console.WriteLine($"  Id: {dto.Id}");
        Console.WriteLine($"  FullName: {dto.FullName}");
        Console.WriteLine($"  Email: {dto.Email}");
        Console.WriteLine($"  Age: {dto.Age}");
        Console.WriteLine($"  PasswordHash: '{dto.PasswordHash}' (should be empty)");
        Console.WriteLine($"  IsActive: {dto.IsActive}");
        
        // Map List
        var entities = Enumerable.Range(1, 5).Select(i => new UserEntity
        {
            Id = i,
            FirstName = $"User{i}",
            LastName = "Test",
            EmailAddress = $"user{i}@example.com",
            IsActive = i % 2 == 0,
            BirthDate = new DateTime(1990 + i, 1, 1)
        });
        
        var dtos = ObjectMapper.MapList<UserEntity, UserDto>(entities).ToList();
        Console.WriteLine($"\nMapped {dtos.Count} users:");
        foreach (var d in dtos)
        {
            Console.WriteLine($"  {d.Id}: {d.FullName} ({d.Email}) - Active: {d.IsActive}");
        }
        
        // Performance test
        Console.WriteLine("\n--- Performance Test ---");
        var sw = System.Diagnostics.Stopwatch.StartNew();
        
        const int count = 100_000;
        for (int i = 0; i < count; i++)
        {
            var _ = ObjectMapper.Map<UserEntity, UserDto>(entity);
        }
        
        sw.Stop();
        Console.WriteLine($"Mapped {count:N0} objects in {sw.ElapsedMilliseconds}ms");
        Console.WriteLine($"Rate: {count / sw.Elapsed.TotalSeconds:N0} maps/sec");
    }
}
```

---

## Exercises

### Exercise 1: Dependency Injector
```csharp
// TODO: สร้าง simple DI container ด้วย Reflection
public class SimpleContainer
{
    // Register: บอกว่า interface -> implementation
    public void Register<TInterface, TImplementation>()
        where TImplementation : TInterface { }
    
    // Register singleton
    public void RegisterSingleton<TInterface, TImplementation>()
        where TImplementation : TInterface { }
    
    // Resolve: สร้าง instance พร้อม inject dependencies อัตโนมัติ
    public T Resolve<T>()
    {
        // ใช้ Reflection หา constructor ที่เหมาะสม
        // สร้าง instances ของ dependencies
        // Inject ผ่าน constructor
        throw new NotImplementedException();
    }
}

// Test:
// container.Register<ILogger, ConsoleLogger>();
// container.Register<IUserRepository, DatabaseUserRepository>();
// container.Register<UserService, UserService>();
// var service = container.Resolve<UserService>();
```

### Exercise 2: Event Bus
```csharp
// TODO: สร้าง event bus ด้วย Reflection
// Handler methods ที่มี [EventHandler] attribute จะถูก register อัตโนมัติ

[AttributeUsage(AttributeTargets.Method)]
public class EventHandlerAttribute : Attribute { }

public class EventBus
{
    // Scan type และ register methods ที่มี [EventHandler]
    public void Register(object handler) { }
    
    // Publish event ไปยัง handlers ที่รับ event type นั้น
    public void Publish<TEvent>(TEvent @event) { }
}
```

### Exercise 3: JSON Serializer
```csharp
// TODO: สร้าง simple JSON serializer ด้วย Reflection
public class SimpleJson
{
    public static string Serialize(object obj)
    {
        // ใช้ Reflection อ่าน public properties
        // แปลงเป็น JSON string
        // Support: string, number, bool, null, nested objects, arrays
        throw new NotImplementedException();
    }
    
    public static T? Deserialize<T>(string json) where T : new()
    {
        // Parse JSON string
        // ใช้ Reflection set properties
        throw new NotImplementedException();
    }
}
```

---

## สรุป

✅ **Reflection** ช่วยให้ตรวจสอบและใช้งาน types ในขณะ runtime

✅ **typeof()** ใช้ compile-time, **GetType()** ใช้กับ instances, **Type.GetType()** ใช้กับ string

✅ **BindingFlags** ควบคุมว่าจะดู members ประเภทใด (public/private, static/instance)

✅ **Invoke()** เรียก method แบบ dynamic, **CreateInstance()** สร้าง object แบบ dynamic

✅ ใช้ **Compiled Expressions** แทน Reflection Invoke เพื่อ performance ดีขึ้น

✅ **Cache** ผลลัพธ์จาก Reflection เพื่อไม่ต้อง reflect ซ้ำ

✅ Use cases หลัก: **Plugin systems**, **ORM**, **DI containers**, **Object mappers**

✅ Reflection มี **overhead** - ใช้เฉพาะเมื่อจำเป็น และ cache ผลลัพธ์

---

## Part ถัดไป

➡️ **Part 036**: Tuples และ ValueTuples - Named elements, Destructuring, Pattern matching กับ tuples

---

*Part 035/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
