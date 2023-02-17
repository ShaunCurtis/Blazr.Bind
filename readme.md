
.Net7.0 introduced another way to do component binding in Blazor.  While, it was done with good intentions, It's probably mor confusing than ever to the people who it is aimed at.

Binding is the way Blazor establishes a two way communication channel with an edit component.  Normally that will be `input` or similar html element, but could be a more exotic edit mechanism - we'll look at one later.

### The InputBase Control Set

Blazor ships with a set of input controls: all are based on `InputBase`.

All implement three Parameters:

1. `Value` is the "in" value for the control. It's strongly typed and the control manages type convertion to the underlying html element.
2. `ValueChanged` is the "out" value for the control.  It's a callback with a strongly typed argument value.  The type convertion from the underlying html element value to the return value is handled internally by the control.
3. `ValueExpression` is a `Func` delegate that defines the actual model object/property. It's used internally to create a FieldIdentifier object, which is used to identify the property in the EditContext state and ValidationMessageStore.

The naming convention here is important.  The "Value" in `@bind-Value` refers to the common name used in the three parameters. You can use another name, but what's the point: stick to conventions.

So when you define this, what is happening?

```csharp
<InputCheckbox class="form-check" @bind-Value=this.model.Value />
```

The first thing to undertstand is that `@bind-Value` is Razor language *syntactic sugar*.  It's just a quick way of defining a binding.

Under the hood this gets compiles into this in the C# file the Razor compiler produces.

```csharp
__builder.OpenComponent<InputCheckbox>(5);
__builder.AddAttribute(6, "class", "form-check");
__builder.AddAttribute(7, "Value", RuntimeHelpers.TypeCheck<Boolean>(this.model.Value));
__builder.AddAttribute(8, "ValueChanged", RuntimeHelpers.TypeCheck<EventCallback<Boolean>>(EventCallback.Factory.Create<Boolean>(this, RuntimeHelpers.CreateInferredEventCallback(this, __value => this.model.Value = __value, this.model.Value))));
__builder.AddAttribute(9, "ValueExpression", RuntimeHelpers.TypeCheck<Expression<Func<Boolean>>>(() => this.model.Value));
__builder.CloseComponent();
```

You can clearly see the Razor compiler is building out the plumbing between the model property and the three Value parameters.

We could construct our code similarily.

```csharp
<InputCheckbox class="form-check"
               Value=this.model.Value
               ValueChanged="(value) => this.model.Value = value"
               ValueExpression="() => this.model.Value" />
```

We can also doing this which gives us the opportunity to do other things when a value is set.

```csharp
<InputCheckbox class="form-check"
               Value=this.model.Value
               ValueChanged=this.OnValueChangedAsync
               ValueExpression="() => this.model.Value" />

//....

    private string message = "No Message";

    private Task OnValueChangedAsync(bool value)
    {
        this.model.Value = value;
        // You can do other stuff here if you need to
        this.message = $"Set at {DateTime.Now.ToLongTimeString()}";
        return Task.CompletedTask;
    }
```

## The New Get/Set

Net7.0 introduced the Get/Set property type concept.  

1. `input-Value:get` provides a strongly typed getter to assign the value to the cntrol.
2. `inout-Value:set` provides a strongly type setter to set the parent property to the value provided by the control.
3. `inpout-Value:after` provides an additional callback that happens after the value has been set.

There are some serious qualifications you need to understand when using these three.

You can't use them together.  This code won't compile.  We'll look at why shortly. 

```csharp
<InputText class="form-control"
           @bind-Value:get=this.model.Name
           @bind-Value:set=this.OnNameChangedAsync
           @bind-Value:after=this.NameChangedAsync
           />
```

This is the normal way to implement the `after`.  Note the method pattern that you need to implement.  You can also use a void (but I no longer consider that good coding practice to promote in event handlers).

```csharp
<InputText class="form-control"
           @bind-Value=this.model.Name 
           @bind-Value:after=this.NameChangedAsync
           />
//....

    private Task NameChangedAsync()
    {
        // You can do other stuff here if you need to
        this.message = $"Set at {DateTime.Now.ToLongTimeString()}";
        Task.CompletedTask;
    }

```