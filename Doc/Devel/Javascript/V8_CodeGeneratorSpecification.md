Javascript: Specification of a Code Generator for V8
====================================================

The aim of this is to evolve a specification for a code generator.

## Top Level structure

The generated code consists of the following blocks:

~~~~
<HELPER_FUNCTIONS>

<INCLUDES>

<CLASS_TEMPLATES>

<FUNCTION_WRAPPERS>

<INITIALIZER>

~~~~

- `HELPER_FUNCTIONS`: static, from swg-file
- `INCLUDES`: static, module property
- `CLASS_TEMPLATES`: dynamically growing, on class declarations
- `FUNCTION_WRAPPERS`: dynamically growing, on method declarations
- `INITIALIZER`: dynamically growing, aggregates everything (to be specified in more detail)

## INCLUDES

~~~~
#include <v8.h>

<USER_DEFINED_INCLUDES>
~~~~

`USER_DEFINED_INCLUDES`: a module property

## CLASS_TEMPLATES

Static references to class templates which are (should be) read-only and can be reused.

~~~~
v8::Persistent<v8::FunctionTemplate> SWIGV8_$CLASSNAME;
~~~~

Notes:
 - it is very important to consider namespaces from the beginning.
   `CLASSNAME` should be fully canonical, e.g., `foo_bar_MyClass` for `foo::bar::MyClass`
 - namespaces do not need a function template, as they will not be
   instantiated

## FUNCTION_WRAPPERS

There are different types of function wrappers:

 - Static Functions (global/namespace/class) 
 - Constructors / Destructors
 - Getters / Settters
 - Member Functions

## Static Functions

TODO

## Constructors

~~~~
v8::Handle<v8::Value> $CLASSNAME_new(const v8::Arguments& args) {
    v8::HandleScope scope;
    
    // retrieve the instance
    v8::Handle<v8::Object> self = args.Holder();
    
    // TODO: declare arguments

    //convert JS arguments to C++
    // TODO: apply input typemaps

    $CLASS_LVAL* ptr = new $CLASS_LVAL($ARGS);
    self->SetInternalField(0, v8::External::New(ptr));
    
    return self;        
}
~~~~

- `$CLASS_LVAL`: should be the canonical name of the class, e.g. `ns1::ns2::MyClass`
- `$ARGS`: arguments should be declared at the beginning and checked and set in the
   input typemap block

## Destructors

TODO

## Getters

~~~~
v8::Handle<v8::Value> $CLASSNAME_get$VARNAME(v8::Local<v8::String> property, const v8::AccessorInfo& info) {
    v8::HandleScope scope;

    // retrieve pointer to C++-this
    $CLASS_LVAL* self = SWIGV8_UnwrapThisPointer<$CLASS_LVAL>(info.Holder());
    $RET_LVAL retval = self->$VARNAME;
    
    v8::Handle<v8::Value> ret = <output_mapping(retval)>;
    return scope.Close(ret);
}
~~~~

- TODO: output typemapping

## Setters

~~~~
void $CLASSNAME_set$VARNAME(v8::Local<v8::String> property, v8::Local<v8::Value> value, const v8::AccessorInfo& info) {
    v8::HandleScope scope;

    // retrieve pointer to this
    $CLASS_LVAL* self = SWIGV8_UnwrapThisPointer<$CLASS_LVAL>(info.Holder());

    // TODO: declare input arg
    
    // map java-script type to C++ type    
    self->$VARNAME = <input_typemapping>;

}
~~~~

TODO: 
 - input type mapping should be addressed in extra step, together with type check

## Member functions

~~~~
v8::Handle<v8::Value> $CLASSNAME_$FUNCTIONNAME(const v8::Arguments& args)
{
    v8::HandleScope scope;

    $CLASS_LVAL* self = SWIGV8_UnwrapThisPointer<$CLASS_LVAL>(args.Holder());

    // TODO: declare input args

    //convert JS arguments to C++
    // TODO: input typemappings and type checking

    //call C++ method
    $RET_LTYPE result = self->$FUNCTIONNAME($ARGS);

    //convert C++ JS arguments to C++
    v8::Handle<v8::Value> ret = <output_typemapping>;

    return scope.Close(ret);
}
~~~~

- if the function does not have a return value, return v8::Undefined

## Initializer

~~~~
void $MODULENAME_Initialize(v8::Handle<v8::Context> context)
{
    v8::HandleScope scope;

    // register the module in globale context
    v8::Local<v8::Object> global = context->Global();

    <CREATE_NAME_SPACES>
    
    <CREATE_CLASS_TEMPLATES>

    <ADD_CLASS_METHODS>

    <INHERITANCES>

    <REGISTER_CLASSES>
}
~~~~

## Namespaces

Namespaces are objects without class templates. I.e., instances are created,
referenced locally, used as contexts for other registrations, and stored
in the according parent contexts.

## Create Class Template

~~~~
    SWIGV8_$CLASSNAME = SWIGV8_CreateClassTemplate("$LOCAL_CLASSNAME" , $CLASSNAME_new);
~~~~

- `LOCAL_CLASSNAME`: the class name without context, i.e., namespaces

## Add Member Function

~~~~
    SWIGV8_AddClassMethod(SWIGV8_$CLASSNAME, "$METHODNAME", $METHOD_WRAPPER);
~~~~

- `METHODNAME`: the name of the function as in C++
- `METHOD_WRAPPER`: the name of the generated wrapper function
   TODO: specify different versions

## Add Property

~~~~
    SWIGV8_AddProperty(SWIGV8_$CLASSNAME, "$VARNAME", $GET_WRAPPER, $SET_WRAPPER);
~~~~

- `GET_WRAPPER`: the name of the generated wrapper for the property getter
- `SET_WRAPPER`: the name of the generated wrapper for property setter; optional (i.e., maybe `NULL`)

## Inheritance

~~~~
    SWIGV8_$CLASSNAME->Inherit(SWIGV8_$SUPERCLASS);
~~~~

- Note: multiple inheritance is not possible; thus we will always take the first parent class

## Register class

~~~~
    $CONTEXT->Set(v8::String::NewSymbol("$LOCAL_CLASSNAME"), SWIGV8_$CLASSNAME->GetFunction());
~~~~

- Note: every class template has an associated ctor function wrapper, which is registered here
- `CONTEXT`: either global, or the according namespace instance

## HELPER_FUNCTIONS

A lot of boiler-plate code can be shifted into static helper functions:

~~~~

/**
 * Creates a class template for a class without extra initialization function. 
 */
v8::Persistent<v8::FunctionTemplate> SWIGV8_CreateClassTemplate(const char* symbol) {
    v8::Local<v8::FunctionTemplate> class_templ = v8::FunctionTemplate::New();
    class_templ->SetClassName(v8::String::NewSymbol(symbol));

    v8::Handle<v8::ObjectTemplate> inst_templ = class_templ->InstanceTemplate();
    inst_templ->SetInternalFieldCount(1);

    return v8::Persistent<v8::FunctionTemplate>::New(class_templ);
}

/**
 * Creates a class template for a class with specified initialization function. 
 */
v8::Persistent<v8::FunctionTemplate> SWIGV8_CreateClassTemplate(const char* symbol, v8::InvocationCallback _func) {
    v8::Local<v8::FunctionTemplate> class_templ = v8::FunctionTemplate::New(_func);
    class_templ->SetClassName(v8::String::NewSymbol(symbol));

    v8::Handle<v8::ObjectTemplate> inst_templ = class_templ->InstanceTemplate();
    inst_templ->SetInternalFieldCount(1);

    return v8::Persistent<v8::FunctionTemplate>::New(class_templ);
}

/**
 * Sets the pimpl data of a V8 class. 
 */
v8::Handle<v8::Value> V8GeneratorUtils::SetInstance(const v8::Arguments& args, void* data) {
    v8::HandleScope scope;

    v8::Handle<v8::Object> self = args.Holder();
    self->SetInternalField(0, v8::External::New(data));

    return self;
}

/**
 * Registers a class method with given name for a given class template. 
 */
void V8GeneratorUtils::AddClassMethod(v8::Handle<v8::FunctionTemplate> class_templ, const char* symbol, v8::InvocationCallback _func) {
    v8::Handle<v8::ObjectTemplate> proto_templ = class_templ->PrototypeTemplate();
    proto_templ->Set(v8::String::NewSymbol(symbol), v8::FunctionTemplate::New(_func));    
}

/**
 * Registers a class property with given name for a given class template. 
 */
void V8GeneratorUtils::AddProperty(v8::Handle<v8::FunctionTemplate> class_templ, const char* varname, v8::AccessorGetter getter, v8::AccessorSetter setter) {
    v8::Handle<v8::ObjectTemplate> proto_templ = class_templ->InstanceTemplate();
    proto_templ->SetAccessor(v8::String::New(varname), getter, setter);
}

~~~~

# Control flow analysis

## Global variables

### Example:
~~~~
int x;

namespace foo {
    double y;
}
~~~~

### Control flow:
Command:
~~~~
swig -analyze -c++ functionWrapper +before globalvariableHandler +before example.i
~~~~

~~~~
enter top() of example
enter variableHandler() of x
enter globalvariableHandler() of x
    | sym:name     - "x"
    | name         - "x"
    | type         - "int"

    enter variableWrapper() of x
    
        enter functionWrapper() of x
            | name         - "x"
            | sym:name     - "x_set"
            | parms        - int
            | wrap:action  - "x = arg1;"
            | type         - "void"
        exit functionWrapper() of x

        enter functionWrapper() of x
            | name         - "x"
            | sym:name     - "x_get"
            | type         - "int"
        exit functionWrapper() of x
    exit variableWrapper() of x
exit globalvariableHandler() of x
exit variableHandler() of x

enter variableHandler() of foo::y
enter globalvariableHandler() of foo::y
    | sym:name     - "y"
    | name         - "foo::y"
    | type         - "double"

    enter variableWrapper() of foo::y
        enter functionWrapper() of foo::y
            | name         - "foo::y"
            | sym:name     - "y_set"
            | parms        - double
            | wrap:action  - "foo::y = arg1;"
            | type         - "void"
        exit functionWrapper() of foo::y
        
        enter functionWrapper() of foo::y
            | name         - "foo::y"
            | sym:name     - "y_get"
            | wrap:action  - "result = (double)foo::y;"
            | type         - "double"
        exit functionWrapper() of foo::y
    exit variableWrapper() of foo::y
exit globalvariableHandler() of foo::y
exit variableHandler() of foo::y
exit top() of example

~~~~

## Simple class

### Example:

~~~~
class A {
public:
    A();
    ~A();
};

namespace foo {
    class B {
    };
}
~~~~

### Control flow:

~~~~
enter top() of example
    enter classHandler() of A
    enter constructorHandler() of A
        enter functionWrapper() of A
            | name         - "A"
            | sym:name     - "new_A"
            | access       - "public"
            | wrap:action  - "result = (A *)new A();"
            | type         - "p.A"
        exit functionWrapper() of A
    exit constructorHandler() of A
    enter destructorHandler() of ~A
        enter functionWrapper() of ~A
            | name         - "~A"
            | sym:name     - "delete_A"
            | parms        - A *
            | wrap:action  - "delete arg1;"
        exit functionWrapper() of ~A
    exit destructorHandler() of ~A
exit classHandler() of A
enter classHandler() of foo::B
    enter constructorHandler() of foo::B::B
        enter functionWrapper() of foo::B::B
            | name         - "foo::B::B"
            | sym:name     - "new_B"
            | access       - "public"
            | wrap:action  - "result = (foo::B *)new foo::B();"
            | type         - "p.foo::B"
        exit functionWrapper() of foo::B::B
    exit constructorHandler() of foo::B::B
    enter destructorHandler() of foo::B::~B
        enter functionWrapper() of foo::B::~B
            | name         - "foo::B::~B"
            | sym:name     - "delete_B"
            | parms        - foo::B *
            | wrap:action  - "delete arg1;"
            | type         - "void"
        exit functionWrapper() of foo::B::~B
    exit destructorHandler() of foo::B::~B
exit classHandler() of foo::B
exit top() of example

~~~~

### Example

~~~~
class A {
public:
    int x;
};
~~~~


### Control flow

~~~~
enter top() of example
enter classHandler() of A
    enter variableHandler() of x
        enter membervariableHandler() of x
            enter functionWrapper() of x
                | name         - "x"
                | sym:name     - "A_x_set"
                | access       - "public"
                | parms        - A *,int
                | wrap:action  - "if (arg1) (arg1)->x = arg2;"
                | type         - "void"
                | memberset    - "1"
            exit functionWrapper() of x
            enter functionWrapper() of x
                | name         - "x"
                | sym:name     - "A_x_get"
                | access       - "public"
                | parms        - A *
                | wrap:action  - "result = (int) ((arg1)->x);"
                | type         - "int"
                | memberset    - "1"
                | memberget    - "1"
            exit functionWrapper() of x
        exit membervariableHandler() of x
    exit variableHandler() of x    
    enter constructorHandler() of A::A
        enter functionWrapper() of A::A
        exit functionWrapper() of A::A
    exit constructorHandler() of A::A
    enter destructorHandler() of A::~A
        enter functionWrapper() of A::~A
        exit functionWrapper() of A::~A
    exit destructorHandler() of A::~A
exit classHandler() of A
exit top() of example
~~~~

## Class method

### Example

~~~~
class A {
public:
    void foo(int x, double y);
private:
    void bar();
};
~~~~

### Control flow

~~~~
enter top() of example
enter classHandler() of A
    enter functionHandler() of foo
        enter memberfunctionHandler() of foo
            enter functionWrapper() of foo
                | name         - "foo"
                | sym:name     - "A_foo"
                | access       - "public"
                | parms        - A *,int,double
                | wrap:action  - "(arg1)->foo(arg2,arg3);"
                | type         - "void"
            exit functionWrapper() of foo
        exit memberfunctionHandler() of foo
    exit functionHandler() of foo
...
exit classHandler() of A
exit top() of example
~~~~

## Static class variable and method

### Example

~~~~
class A {
public:
    static int x;
    
    static void foo();
};
~~~~

### Control flow

~~~~
enter top() of example
enter classHandler() of A
    enter variableHandler() of x
        enter staticmembervariableHandler() of x
            enter variableWrapper() of A::x
                enter functionWrapper() of A::x
                    | name         - "A::x"
                    | sym:name     - "A_x_set"
                    | parms        - int
                    | wrap:action  - "A::x = arg1;"
                    | type         - "void"
                exit functionWrapper() of A::x
                enter functionWrapper() of A::x
                    +++ cdecl ----------------------------------------
                    | name         - "A::x"
                    | ismember     - "1"
                    | sym:name     - "A_x_get"
                    | variableWrapper:sym:name - "A_x"
                    | wrap:action  - "result = (int)A::x;"
                    | type         - "int"
                exit functionWrapper() of A::x
            exit variableWrapper() of A::x
        exit staticmembervariableHandler() of x
    exit variableHandler() of x
    enter functionHandler() of foo
        enter staticmemberfunctionHandler() of foo
            enter globalfunctionHandler() of A::foo
                enter functionWrapper() of A::foo
                    | name         - "A::foo"
                    | sym:name     - "A_foo"
                    | wrap:action  - "A::foo();"
                    | type         - "void"
                exit functionWrapper() of A::foo
            exit globalfunctionHandler() of A::foo
        exit staticmemberfunctionHandler() of foo
    exit functionHandler() of foo
    enter constructorHandler() of A::A
    exit constructorHandler() of A::A
    enter destructorHandler() of A::~A
    exit destructorHandler() of A::~A
exit classHandler() of A
exit top() of example
~~~~

## Inheritance

### Example

~~~~
class A {
};

class B: public A {
};
~~~~

### Control flow

~~~~
enter top() of example
enter classHandler() of A
    +++ class ----------------------------------------
    | name         - "A"
    | sym:name     - "A"
...
exit classHandler() of A
enter classHandler() of B
    | name         - "B"
    | sym:name     - "B"
    | privatebaselist - 0xf1238f10
    | protectedbaselist - 0xf1238ef0
    | baselist     - 0xf1238ed0
    | bases        - 0xf1239830
    | allbases     - 0xf1239c30
...
exit classHandler() of B
exit top() of example
~~~~