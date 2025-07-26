# Understanding the Compilation and Execution Phases of Static and Instance Class Members in C#

## How Static and Instance Members Are Compiled

### **1. Compilation of Empty Classes**
#### A brief information about C# compilers

- **Roslyn** - Roslyn is the official C# compiler platform from Microsoft. It powers the csc compiler and the C# compilation process.
- **csc** - `csc` is the C# command-line compiler executable that uses the Roslyn compiler platform to compile C# source code into assemblies.
    - In other words, (`csc.exe`) compiles C# source code files into assemblies (`.dll` or `.exe`).
    - It can't run the assemblies automatically, in order to run them you either;
        - Run the `.exe` file directly (if it’s a console or desktop app), or 
        - Use dotnet yourapp.dll (for .NET Core apps built as DLLs).  
- **dotnet vs csc** -  `csc` is the low-level C# compiler tool while dotnet is a high-level CLI that manages building, running, and more. However dotnet uses `csc` under the hood to compile C# code when needed.

Code example 1;
```c#
public class Person{
}
```
Code example 2;
```c#
public static class Person{
}
```

#### Compilation of code examples

1. Roslyn parses the code into IL (MSIL) for both classes.

    - Code example 1 (Simplified IL code);
      
          
             .class public auto ansi beforefieldinit Person
                extends [System.Runtime]System.Object
             {}
          
         
    - Code example 2 (Simplified IL code);
      
         
            .class public abstract sealed auto ansi beforefieldinit Person
              extends [System.Runtime]System.Object
            {}
             

2. Roslyn creates metadata for classes (even if they are empty; metadata is always created for any type). This metadata includes;

    - Type name (Person - in our case),

    - Namespace (If declared within one),

    - Type kind (It’s a class, not a struct, enum, etc.),

    - Modifiers (public, sealed, abstract, etc. if used),

    - Base type (Defaults to System.Object if not explicitly defined),

    - Assembly (The assembly in which the type is defined),

    - Custom attributes (Empty if none are declared, otherwise included),

    - Method table entries (At least the inherited System.Object methods like ToString() etc.)

3. Roslyn stores members of classes' under it's type definition. If there is any.

        Code example 1 (Simplified IL code);

        .class public auto ansi beforefieldinit Person
               extends [System.Runtime]System.Object
        {
            "this is type's scope"
        }
        
        Code example 2 (Simplified IL code);

        .class public abstract sealed auto ansi beforefieldinit Person
               extends [System.Runtime]System.Object
        {
            "this is type's scope"
        }

        Both of *instance* and *static* members goes inside the type's scope.

4. The output is saved in `.dll` or `.exe` file and these compiled .dll and .exe files contains;
 
    - IL code (Intermediate Language)

    - Metadata tables (TypeDef, MethodDef, etc.)

    - PE (Portable Executable) headers

    - Manifest

`.dll` (just one physical file) and `.exe` (just one physical file) files are stored in the SSD (non-volatile storage). They are not executable directly by the CPU.
<br/>

5. Both `static classes` and `instance classes` are stored in the IL code. But there are some differences and similarities between them.
  - **SIMILARITIES**
    - Both static classes and instance classes have only one shared type definition in the assembly (`.dll` or `.exe`).
- **DIFFERENCES**
    - **Static Classes**:
        -  A static class’s type definition is shared across the entire application. All static members (fields, methods) are stored once 
        in memory and shared across the entire AppDomain.

        - No object instantiation is required.

        - Every call like `Person.Speak()` uses the same metadata and method address resolved from a single MethodTable.

        - Method IL instruction is `call`.

        - Method lookup is via type metadata (MethodTable).

        - No memory allocation occurs except for the stack during the call.

        - There’s no need for a `this` pointer since there’s no object instance or corresponding memory address.

        - All members are marked as `static` in IL code.

        - If there is no explicit constructor, but a static field with an initializer exists, the compiler generates a private static constructor (to prevent instantiation).

        - If there is no explicit constructor and no static field with an initializer, the compiler does not generate a constructor. Therefore, for an empty static class without an explicit constructor, the compiler will not create an implicit constructor.

    - **Instance Classses**:
        - While an instance class also has a single shared type definition, every object you create from it allocates separate memory in the heap for its layout and also fields.

        - Fields are copied per object, giving each object its own state. This does not mean copying the same value, but rather allocating separate memory addresses for each object's fields.

        - Method calls like `person.Speak()` are resolved through the shared MethodTable, but they operate on object-specific data. The methods are shared, not copied.

        - Method IL instruction is `callvirt`.

        - An object is needed to call a method.

        - Method lookup is performed via the object's MethodTable pointer.

        - Memory allocation is needed (on the heap for the object).

        - The `this` pointer is passed (implicitly) as a hidden first parameter in the methods. 
        
        - A default parameterless constructor is automatically generated if no constructor is defined.
        
        **Note:**
        - **Implicit this pointer:**

            - *Compile-time:* The `this` reference is implicitly passed to the method, including constructors, at compile-time. This is because the compiler knows that the constructor or instance method is operating on a specific object instance.

            - *Runtime:* When the constructor executes at runtime, the `this` pointer refers to the object instance being created or modified. Furthermore, `this` is a pointer to the memory address of the object instance on the heap — it points to the start of the object’s memory layout. It is different from the MethodTable pointer.

        - **Explicit this pointer:**
            - *Compile-time:* You explicitly use `this` when referring to the current object. For example, `this.Name = name;` in the constructor is simply telling the compiler that you're setting the `Name` field of the object being constructed.

            - *Runtime:* During runtime, `this` still points to the current object, but it's used explicitly by the programmer. The behavior is exactly the same as the implicit `this`.
          

### **2. Compilation of Classes with Members**

Code example 1 ;
```c#
public class Person{
        private const string IdentityNumber = 12345678;
        public string Name;
        public string Email{get; set;}
        public void Speak(){
                Console.WriteLine($"Hello my name is; {Name}");
        }
}
```
 Code example 2 ;
```c#
public static class Person{
        private static const string IdentityNumber = 12345678;
        public static string Name;
        public static string Email{get; set;}
        public static void Speak(){
                Console.WriteLine($"Hello my name is; {Name}");
        }
}
```

1. Roslyn parses the code to IL (MSIL) code for both examples.
    - Code example 1 (Simplified IL code);

    ```
        .class public auto ansi beforefieldinit Person
        extends [System.Runtime]System.Object
        {
         .field private static literal string IdentityNumber = "12345678" //         const becomes literal + static
         .field public string Name
         .property instance string Email()
         {
            .get instance string Person::get_Email()
            .set instance void Person::set_Email(string)
         }

         .method public hidebysig instance void Speak() cil managed 
         {
            // body with Console.WriteLine
         }

         .method public hidebysig specialname instance string get_Email() cil        managed
         {
            // backing field return
         }

         .method public hidebysig specialname instance void set_Email(string)        cil managed
         {
             // backing field assignment
         }
        }
    ```
    - Code example 2 (Simplified IL code);

    ```
        .class public abstract sealed auto ansi beforefieldinit Person
        extends [System.Runtime]System.Object
        {
         .field private static literal string IdentityNumber = "12345678" // const still literal
         .field public static string Name
         .property string Email()
         {
             .get string Person::get_Email()
             .set void Person::set_Email(string)
         }

         .method public hidebysig static void Speak() cil managed
         {
                // body with Console.WriteLine
         }

         .method public hidebysig specialname static string get_Email() cil  managed
         {
                // return static backing field
         }

         .method public hidebysig specialname static void set_Email(string)  cil managed
         {
                // assign to static backing field
         }
        }
    ```
        
2. Whether declared as static or instance, a constant is fully resolved at compile time. The compiler replaces every usage of a `const` with its literal value in the IL.

3. Whether declared as static or instance, properties have backing fields for assignment and retrieval in the IL code.

4. Roslyn creates metadata for these examples.

5. The output is saved in `.dll` or `.exe` file.

## What is a .dll or .exe file - also known as a PE (Portable Executable) file - in .NET? 

Here is an image of a simlified PE file;
<p align="center">
<img width="832" height="520" alt="Assembly" src="https://github.com/user-attachments/assets/dba456df-4d36-4fae-9023-b7ad38d30fd5" />
</p>


**Breakdown of main sections :**

1. MS-DOS Header

    - Just a stub. Exists for backwards compatibility.

    - Says “This program cannot be run in DOS mode.”

2. PE Header

    - Standard Windows PE header.

    - Describes file type, machine type, section sizes.

3. CLI Header (Common Language Infrastructure)

    Marks this file as a .NET assembly.

    Points to:

    - IL Code

    - Metadata

    - Entry point (e.g. Main())

4. .text Section

    - Contains IL code generated by Roslyn.

    - Not native machine code yet — JIT will convert it later.

## Metadata Tables in More Detail
In a compiled .NET assembly (`.dll` or `.exe`), there is a section of the file called the metadata and this compilation happens in the build time. Metadata includes structured tables, very much like database tables — meaning:

- Rows represent entities (types, methods, fields, etc.).

- Columns represent attributes (e.g., name, type, flags, tokens).

- Each row has a token, like a primary key (e.g., 0x02000002 means second row in the TypeDef table).

These tables are defined by the ECMA-335 CLI specification, which governs how .NET assemblies are structured.
<br/>

The CLI defines (up to) 64 metadata tables, including reserved gaps—common ones include:

        1 (0x1): TypeRef  
        0 (0x0): Module  
        2 (0x2): TypeDef  
        3 (0x3): FieldPtr  
        4 (0x4): Field  
        5 (0x5): MethodPtr  
        6 (0x6): Method (MethodDef)  
        7 (0x7): ParamPtr  
        8 (0x8): Param  
        9 (0x9): InterfaceImpl  
        A (0xA): MemberRef  
        B (0xB): Constant  
        C (0xC): CustomAttribute  
        D (0xD): FieldMarshal  
        E (0xE): DeclSecurity  
        F (0xF): ClassLayout  
        10 (0x10): FieldLayout  
        11 (0x11): StandAloneSig  
        12 (0x12): EventMap  
        13 (0x13): EventPtr  
        14 (0x14): Event  
        15 (0x15): PropertyMap  
        16 (0x16): PropertyPtr  
        17 (0x17): Property  
        18 (0x18): MethodSemantics  
        19 (0x19): MethodImpl  
        1A (0x1A): ModuleRef  
        1B (0x1B): TypeSpec  
        1C (0x1C): ImplMap   
        … (reserved up to 0x3F)



**Here are some of the tables and their raw views**

1. `MethodDef` table in the assembly;
   
    <img width="840" height="248" alt="MethodDefTable" src="https://github.com/user-attachments/assets/e0294fe8-ac05-4cfe-9769-6a0367db2eb0" />

    - cRecs: number of method entries

    - cbRec: size in bytes per row (14)
      
      **Columns:**
      - RVA = IL code address
    
      - ImplFlags = code implementation flags
      
      - Flags = method attributes (e.g., public, virtual)
      
      - Name = index into #Strings
      
      - Signature = index into #Blob
      
      - ParamList = starting index in Param table
    

3. `TypeDef` table in the assembly;
     
     <img width="840" height="248" alt="TypeDefTable" src="https://github.com/user-attachments/assets/475eb3f4-b367-4229-b251-eb906975a12d" />

4. `Field` (table ID 4) is often formatted as:
   
    <img width="840" height="160" alt="FieldTable" src="https://github.com/user-attachments/assets/f51bf6d6-d78e-45a4-b259-b0b26f351744" />

    - Flags: field visibility, static, etc.
  
    - Name: index into #Strings

    - Signature: index into #Blob (type info)


## Runtime Memory Tables in More Detail
When it comes to runtime the CLR loads metadata and IL code (the `.dll` or `.exe`) into RAM (memory).
<br/>

Here is a broad view of the whole compilation / run time flow;
<p align="center">
  <img width="829" height="400" alt="WholeFlow" src="https://github.com/user-attachments/assets/8d2f94eb-2cf3-4481-be14-b843824829f0" />
<p/>
  
**Here are some of the tables**

1. `Method` table in memory

    - A `per-type` structure created and used by the CLR that stores metadata about methods, type hierarchy, interfaces, etc.

    - It is stored in memory not in the disk (SSD).

    - Method bodies are stored in this table.

    - Contains general information about a type (methods, interfaces, vtable pointer, etc.).

    This is how a method table might look like in the memory;

    <img width="870" height="341" alt="MethodTable" src="https://github.com/user-attachments/assets/635c9b77-473d-4245-927a-b6503c485567" />
    <img width="870" height="337" alt="MethodTable02" src="https://github.com/user-attachments/assets/d7fea9e5-12b5-4669-8dc5-2cb6997c6223" />

    - This is just a block of memory with a known layout.

    - It’s called a "table" because it allows fast lookup:

            E.g., methodTable[4] → pointer to method implementation

    - But it’s really a pointer-based structure, not a DB-style table.


2. `VTable` (Virtual Method Table)

    - A runtime table (used especially for polymorphism) that stores function pointers to virtual method implementations for a given type.

    - It is a part (or sub-table) of the method table that specifically deals with virtual method dispatch.

    
3. `Field` Table (Conceptual)

    - Conceptual table that maps field names to their offsets/types. This info is stored in metadata and used at runtime to allocate object memory layout.

    - Again it is not a physical "table" in RAM, but the field layout information is stored in the type's metadata and used when allocating instances.

    - The actual fields (i.e. data) live in:

      - The stack if it's a value type on the stack (e.g., int, struct),

      - The heap if it's part of a reference type instance (e.g., in a class).

**Here is a broad overview**
<p>
  <img width="450" height="425" alt="MemoryTableOverview01" src="https://github.com/user-attachments/assets/fd0657c6-dbfa-4cbc-ba68-b0ecc8a66cc9" />
  <img width="460" height="480" alt="MemoryTableOverview02" src="https://github.com/user-attachments/assets/a92c5a48-ac97-4641-bfcd-2628a6fcc7a9" />
<p/>
  
**And here is a summary about tables and their locations**
<p align="center">
  <img width="1019" height="357" alt="MemoryTableOverviewSummary" src="https://github.com/user-attachments/assets/ea1c27e1-89d4-459d-a85e-3c8931227356" />
<p/>


## Run-Time Execution Flow of Static and Instance Members

### Execution of Instance Members According to the Example Below – Example 1

Code example 1 ;
```c#
public class Person{
        private const string IdentityNumber = 12345678;
        public string Name;
        public string Email{get; set;}
        public void Speak(){
                Console.WriteLine($"Hello my name is; {Name}");
        }
}
```
```c#
var person = new Person();
person.Name = "User";
person.Email = "user@mail"
person.Speak();

```
The compiled .exe or .dll is loaded, and the metadata is checked by the CLR (Common Language Runtime).

1. As there is a main entry point of every program the main method starts execution.

    *Stack Frame;*
   <p>
      <img width="830" height="154" alt="StackFrame01" src="https://github.com/user-attachments/assets/7e04ec23-57d2-4d65-9efc-4a4b3f77c0e9" />
   </p>

```c#
// code
var person = new Person();
```
2. Calls the constructor.
 
    *Stack Frame;*
    <p>
      <img width="830" height="204" alt="StackFrame02" src="https://github.com/user-attachments/assets/05e8cf6f-27b2-476d-9e9d-e3aca7f51bb6" />
    </p>

    - Constructor allocates memory on the managed heap for a Person object.

       **The object layout includes:**

    -  A type pointer (MethodTable) to link to the class's method table.

    - Memory for each instance field: Name and the backing field for Email.   (Initializes fields (to null by default))

    - A reference to the newly created object is stored in the local variable person (on the stack).

    - The constructor returns to Main().

    - ❌ Person..ctor frame is popped.

```c#
// code
person.Name = "User";
```
3. Name is a public field, so this is a direct memory write into the Name field of the person object in the heap.

    - No method call is involved here; the compiler generates direct IL code to set the field.

    - No stack frame is created; the stack frame remains unchanged.

        *Stack Frame;*
      <p>
        <img width="830" height="166" alt="StackFrame03" src="https://github.com/user-attachments/assets/8f77658a-2de3-4eef-bb00-e25707796a40" />
      </p>
```c#
// code
person.Email = "user@mail"
```

4. This line invokes the property setter method set_Email(string) that was automatically generated by the compiler.

    *Stack Frame;*
    <p>
      <img width="830" height="202" alt="StackFrame04" src="https://github.com/user-attachments/assets/7f68394c-c1ef-4bfe-8ffd-44f2d61a164a" />
    </p>
  
    - The method sets the hidden backing field to 'user@mail'.

    - Returns to Main().

    - ❌ set_Email frame is popped.

```c#
// code
person.Speak();
```
5. This calls the Speak() instance method on the person object.

    *Stack Frame;*
    <p>
        <img width="830" height="203" alt="StackFrame05" src="https://github.com/user-attachments/assets/3df7d1fd-7e57-47a6-8215-f993dfe683cb" />
    </p>

    Inside Speak():
    ```c#
    Console.WriteLine($"Hello my name is; {Name}");
    ```
    - `Console.WriteLine(string)` called.

    *Stack Frame;*
    <p>
        <img width="830" height="263" alt="StackFrame06" src="https://github.com/user-attachments/assets/fbbe796d-efe1-4497-afc0-113e5979e606" />
    </p>
    
    - The $"" syntax compiles into a call to string.Format(...) or uses an interpolated string handler (depending on optimization).

    *Stack Frame;*
    <p>
       <img width="830" height="352" alt="StackFrame07" src="https://github.com/user-attachments/assets/1f9b77ab-fe0a-42e4-97cb-b295e4633419" />
    </p>
    
    - The current value of Name ('User') is inserted.

    - string.Format creates "Hello my name is; User" → returns to Console.WriteLine()

    - ❌ string.Format frame is popped.

    - Console.WriteLine, which sends it to output → returns to Speak().

    - ❌ Console.WriteLine frame is popped.

    - Speak() → returns to Main().

    - ❌ Person.Speak frame is popped.

    - Main() → ends program.

    - ❌ Main frame is popped.

### Execution of Static Members According to the Example Below – Example 2

 Code example 2 ;
```c#
public static class Person{
        private static const string IdentityNumber = 12345678;
        public static string Name;
        public static string Email{get; set;}
        public static void Speak(){
                Console.WriteLine($"Hello my name is; {Name}");
        }
}
```
```c#
Person.Name = "User";
person.Email = "user@mail"
person.Speak();
```

Compiled .exe or .dll is loaded and the metadata checked by the CLR (Common Language Runtime).

Since the class contains static fields, the compiler emits a private static constructor (.cctor) to initialize them.

The const field is inlined directly into IL and is not stored as a memory field.

1. As there is a main entry point of every program the main method starts execution.

    *Stack Frame;*
    <p>
      <img width="830" height="154" alt="StackFrame01" src="https://github.com/user-attachments/assets/7e04ec23-57d2-4d65-9efc-4a4b3f77c0e9" />
    </p>
    
2. The static constructor `Person.cctor` does get executed, but it does not typically appear on the visible call stack because:

    - It created internally by the CLR

    - Executed once, just before the first static member access

    - Not visible in most debuggers unless you dig very deep
        ```c#
        // code
        person.Name = "User";
        ```    
        Before this line executes, the CLR does this:

        ```c# 
        if (Person type is not initialized)
            → Run Person..cctor()  ← static constructor
        ```
        So stack would be like this;

        *Stack Frame;*
        <p>
          <img width="830" height="202" alt="StackFrame08" src="https://github.com/user-attachments/assets/a5966076-b72d-451c-ba99-14761a6b0305" />
        </p>
```c#
// code
person.Name = "User";
```
3. No stack frame change occurs because there is just field access, which happens inside the Main method. So the stack remains the same.

   *Stack Frame;*
   <p>
      <img width="830" height="166" alt="StackFrame03" src="https://github.com/user-attachments/assets/8f77658a-2de3-4eef-bb00-e25707796a40" />
   </p>
   
    - Just data written into static memory.

```c#
person.Email = "user@mail"
```
4. Same as above — only sets a static `property. set_Email()` is compiler-generated but inlined by JIT.

    - And yes — `set_Email(string value)` is a method, so a stack frame could be created.

    - But for simple auto-properties, the JIT optimizes them to direct memory store operations.

    - The assignment may not appear on the visible call stack because it's treated like a field write.

    - And the frame is very short-lived — it pushes and pops almost instantly.

```c#
person.Speak();
```
5. Now a method call occurs, so a new stack frame is pushed.

    *Stack Frame;*
    <p>
      <img width="830" height="208" alt="StackFrame09" src="https://github.com/user-attachments/assets/548a3631-f932-4e36-8e42-93eb8bd90e2a" />
    </p>

    And the rest follow in that order.

    *Stack Frames;*
    <p>
      <img width="830" height="265" alt="StackFrame10" src="https://github.com/user-attachments/assets/7ce8efc4-12fd-49e3-a19a-fd488763937a" />
      <img width="830" height="319" alt="StackFrame11" src="https://github.com/user-attachments/assets/3ae6b172-8d22-4926-a924-204ee22121dd" />
    </p>  
    
    - After String.Format returns:

    - ❌ String.Format frame is popped.

    - ❌ Then Console.WriteLine.

    - ❌ Then Person.Speak.

    - ❌ Back to Main, and Main popped.

    - Then the program ends.

<br/>
<br/>

**NOTES:**
1.  Static Indexer ❌
   
```c#
public static string this[int index] // ❌ Not allowed
```
Why not?

- Indexers require an instance to be meaningful. They are syntactic sugar for obj[index].

- Static classes or members don't refer to instances, so there's no this context.

- Since indexers are tied to an object, not a class, they can't be static.

Use a static method instead, like GetByIndex(int index).

2. Static Enum ❌
   
```c#
public static enum Color { Red, Green, Blue } // ❌ Not allowed
```
Why not?

- enum in C# is already a static-like value type. You don't create instances of enums.

- Declaring an enum as static is redundant and not allowed by the compiler.

- Enums automatically behave like static members:
        You can use Color.Red without creating a Color object.

Enums are implicitly static-like already. You don’t need to mark them as static.

3. Static Destructor ❌
   
```c#
public static ~Person() // ❌ Not allowed
```
Why not?

- Destructors (finalizers) run when an object is garbage collected.

- Static classes/members are not tied to any object and live until the end of the app domain.

- There's no instance to clean up, so static destructors make no sense.

The runtime handles cleanup of static data when the app ends — no destructor needed.

4.  Static Delegate ❌
   
```c#
public static delegate void MyHandler(); // ❌ Compiler error
```
Why not?

- delegate is a type definition, not an instance or method.

- In C#, type definitions (like class, struct, delegate, enum) define blueprints. static is only meaningful when applied to a concrete type (like a class) or members inside a type.

  But you can have a static delegate field or event,
  ```c# 
  public class EventBus {
      public static Action OnEvent;
  }
  ```

5. Static Interface ❌

Why not?

- Interfaces do not have implementation logic or state (until default interface methods in C# 8, but still no state).

- static implies no instantiation and shared members, but interfaces are not instantiated anyway.

- Allowing static interface would be confusing and semantically incorrect, as there's no object to be shared or logic to define in a pure interface.

What is allowed?

- Since C# 8, interfaces can contain static members, but the interface type itself still cannot be static:
```c#
public interface IExample {
    static int GetSomething() => 42; // ✅ OK in C# 8+
}
```
6. Static Attribute ❌

Why not?

- Attributes are instantiated by the runtime and attached to types, methods, etc.

What is allowed?

- Attributes must be instantiable reference types, i.e., normal (non-static) classes derived from System.Attribute:
```c#
public class MyAttribute : Attribute {
    public string Name { get; set; }
}
```
7. Static Generic Type Parameters ❌

Why not?
- The compiler does not know what T will be at compile-time.

- You cannot create static members that rely on a type that is unknown at compile time unless it's restricted (e.g., where T : new()).

- Also, every closed generic type gets its own copy of static members, so Container<int> and Container<string> would have different static fields, which breaks the idea of a single static member.

8. Static using and namespace ❌

```c#
using static MyApp.MyClass;
```

- This does not mean that using itself is static — rather, this is a special C# feature introduced in C# 6.0.

- It is a special keyword combination that tells the compiler:

- "Bring in all accessible static members of the specified type into scope."

However the code below is not allowed;
```c#
static using MyList = System.Collections.Generic.List<string>; // ❌ Not allowed
```
Why not?

  - using, namespace, and type aliases are compile-time features — they don’t exist at runtime.

  - static is a runtime concept — it changes how the CLR handles memory and access.
