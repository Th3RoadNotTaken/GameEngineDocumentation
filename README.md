# C++ Game Engine
Modular, data-driven C++ game engine designed for flexibility, performance, and rapid iteration. At its core, the engine exposes a lightweight runtime reflection system, a JSON-based content pipeline, and a service-oriented architecture that lets you plug in or swap out subsystems without touching core code.

You can access the engine documentation [here](https://th3roadnottaken.github.io/GameEngineDocumentation/). The code for my game engine is private, but I can provide access on request.

## Key Architectural Features - 

1. Custom Memory Service
   - Multiple Heaps: Create and manage named heaps with custom sizes
   - Alloc/Free with Coalescing: Merges free blocks to minimize fragmentation
   - Usage & Debug Stats: Query heap usage, count active allocations, and dump allocation records for leak detection
   - Per-Heap Allocators: Obtain lightweight allocators tied to specific heaps for scoped or specialized allocations
    
1. Content Parsing Framework
   - Handles translation of JSON files into in-engine data structures -
     - Core Data Types -
       - Datum: Variant-like container for ints, floats, strings, RTTI*, and nested Scope tables.
       - Scope: A map of named Datum entries, supporting parent-child relationships.
     - IParseHandler -
       - Interface methods: Begin, End, EnterKey, ExitKey. These functions on each handler perform runtime checks on the JSON data type passed, and convert data to C++ if they are capable; otherwise, they hand the execution over to another handler following the chain of responsibility principle.
       - Concrete handlers manage specific JSON constructs (arrays, objects, primitives).
     - Parser -
       - Walks a Json::Value (from JsonCpp) and invokes IParseHandler callbacks.
     - ParseWriter -
       - Maintains a stack of Scope* for nested objects.
       - Builds Datum entries and Scope hierarchies as JSON is traversed.
     - ContentService -
       - Sits above Parser + ParseWriter.
       - Registers file-level IJsonParser implementations by extension (e.g., .json).
       - Reads files, passes JSON to parser/writer, returns a fully populated Scope.
  
2. RTTI and Reflection
   - Implements a minimal RTTI layer (RTTI base class) with macros to declare types
   - RTTI Base Class: All reflected types inherit from RTTI
   - Type Registration: Macros (RTTI_DECLARATIONS, RTTI_DEFINITIONS) register classes and a string-based factory name
   - Capabilities -
     - Type Lookup: RTTI::TypeName() returns a class’s name
     - Safe Casting: As<T>() performs a type check before casting
     - Factory Hooks: Reflection data feeds into the FactoryService

3. Service Manager
   - Central registry that provides access to global services
   - Pattern: Service Locator + Static Auto-Registration
   - Function: At startup, subsystems (e.g., MemoryService, ContentService, FactoryService) self-register with the central registry. Clients request services via the interface -
     ```
     auto* mem = ServiceManager::ProvideInterface<IMemoryService>();
   - Design Highlights -
     - Locator Pattern: Clients request by interface, not by concrete class
     - Lazy Initialization: Services are instantiated on first access or at startup
     - Swapability: Replace one implementation with another with no client-side changes
    
4. Attributed Objects
   - Extends RTTI to allow runtime metadata on objects -
     - Signature entries describe each field’s name, type, and offset
     - Attributed base class -
       - Holds a vector of Signature for class attributes
       - Instances get a parallel list for instance-level attributes
      
5. Actions
   - Coordinates sequenced behaviors via JSON definitions
   - Action -
     - Base class inheriting Attributed
     - Defines a virtual Update method
    - ActionList -
      - Contains a list of actions and invokes them in order
    - Factory-driven Construction -
      - JSON describes an array of actions (type: factory name, plus parameters)
      - The system uses FactoryService to create each Action dynamically
    - Runtime Flow -
      - Parse JSON → Scope
      - For each element, get FactoryService factory → create Action
      - Populate attributes on the Action
      - Execute Action.Update()/ActionList.Update() each frame or on trigger
