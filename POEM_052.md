POEM ID: 052  
Title:  Function based component definition for OpenMDAO  
authors: [justinsgray, Ben Margolis, Kenny Lyons]  
Competing POEMs: N/A    
Related POEMs: 039   
Associated implementation PR:  

##  Status

- [x] Active
- [ ] Requesting decision
- [ ] Accepted
- [ ] Rejected
- [ ] Integrated

## Motivation

Define an OpenMDAO component, including all I/O with metadata, the compute method, and potentially derivatives using a purely function based syntax. 
Why? 
- Make it easier/reduce boiler plate for defining a component
    - generate an XDSM for design reviews and facilitate regression-based spec testing
    - faster to starting point for OpenMDAO-specific configuration
    - iterative OpenMDAO-specific configuration
- Define relevant functions (compute, apply_nonlinear, compute_partials) using standard python `def` syntax
    - original function(s) are still accessible and callable

We should support multiple workflows this way.

An example of code using minimal coupling with the OpenMDAO API: 

```python
# engineering code
def some_component_func(x, y, z):
    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar

# OpenMDAO middleware
SomeComponent = om.ExplicitFuncCompWithOutput(some_component_func, 'foo', 'bar')
```

In this example, the engineering code is not modified at all, `ExplicitFuncCompWithOutput` takes the function and consumes the remaining `*args` as output objects/metadata. Assume just a string is providing just a name, but could be a tuple, dict, etc. Should ultimately define one right way to handle managing IO metadata as it will come up more later. This may be a special use case (and function/interface) but it should return the same type of object/class.

Inputs and outputs without metadata should get defaulted to the least restrictive possible (broadcastable, unitless, etc) 


For model design where the code is being written for an OpenMDAO component, some coupling is acceptable to streamline workflow (reduce repeat code). For example, XDSM to spec test work flow.

```python
def some_component_func(x, y, z) -> ['foo', 'bar']:
    return

SomeComponent = om.ExplicitFuncComp(some_component_func)
```

The resulting `SomeComponent` has inputs and outputs with the least restrictive metadata possible and can be placed in a group so that an XDSM may be drawn and spec test regression data may be generated.
Does OpenMDAO run the compute to test things during setup? does it need to return something?

Next steps include writing the function code and configuring the Component.
Component configuration is what's included in the I/O and "everything else".
For the I/O, I will ignore the fully de-coupled engineering code/ OpenMDAO API case.
For the coupled case, we can do more annotations or decorators.

For the annotations, it could be nice to provide a custom type that takes the metadata fields as var. And read defaults as default value like:

```python
def some_component_func(x: om.func.Input(units='m') = 0., y, z) ->
                        [om.func.Output('foo'),
                         om.func.Output('bar', units='1/m', shape=4)]:
    return

SomeComponent = om.ExplicitFuncComp(some_component_func)
```
Or if it's better to use a dict or 3rd party metadata container, that's fine too. Or support both. 

Or, the input/output can be configured by decorator as proposed by Justin. I would like it if the input decorators inspected the name of the input variable by default. 

For more complicated OpenMDAO API specific calls, perhaps we subclass? Another feature is other functions having a normal signature, either as stand-alone functions or methods of subclass. Could use standard OpenMDAO method name and decorate to handle function input/output. For standalone, may need special API call or flag.
Or are both cases solved by just assuming that signature type for this subclass and wrapping everything with an IO handler.

```python
class SomeComponent(om.ExplicitFuncComp(some_component_func)):
    def __setup_hook__(self):
        """
        can make any any component api calls here. Can use hooks (might want pre/post setup)
        or a user can over-write setup or any other functions but call super().setup() appropriately
        """

     @om.func.process_io # or maybe this is the default behavior for this class?
     def compute_partials(self, x, y, z, J): 

        J['foo', 'x'] = -3*np.log(z)(3*x+2*y)**2 
        J['foo', 'y'] = -2*np.log(z)(3*x+2*y)**2 
        J['foo', 'z'] = 1/(3*x+2*y) * 1/z

        J['bar', 'x'][:] = 2 # need to set all elements of array
        J['bar', 'y'][:] = 1 # need to set all elements of a

```

```python
def J_some_func(x, y, z, J): 

    J['foo', 'x'] = -3*np.log(z)(3*x+2*y)**2 
    J['foo', 'y'] = -2*np.log(z)(3*x+2*y)**2 
    J['foo', 'z'] = 1/(3*x+2*y) * 1/z

    J['bar', 'x'][:] = 2 # need to set all elements of array
    J['bar', 'y'][:] = 1 # need to set all elements of a

class SomeComponent(om.ExplicitFuncComp(some_component_func)):
    def __setup_hook__(self):
        self.compute_partials(J_some_func)

```

Once you have the class, you can do any additional OpenMDAO api calls. In a post setup hook, could probably even manipulate the automatically generated stuff (like from the least specified annotations or even `om.ExplicitFuncCOmpWithOutput`, which should also be subclassable like the other).
This provides some some separation of engineering code and OpenMDAO api, reduces boilerplate, etc as desired.


## Explicit API Description

A purely function based syntax for component definition has several nice properties. 
It offers a fairly compact syntax, especially in cases where there is uniform metadata all I/O in the component. 
It also provides an interface that is much more compatible with algorithmic differentiation than the traditional dictionary-like arguments to the `compute` method of the standard OpenMDAO API. 

The proposed functional component API in this POEM was inspired by the function registration API in POEM_039, 
but seeks to extend that concept much further to allow full component definitions (i.e. more than exec-comps) using nothing more than a python function definition. 
Since Python 3.0, the language has supported function annotations which can be used to provide any and dictionaries of metadata that a special component can interrogate and then wrap in the normal OpenMDAO API. 

Here is a basic example of the proposed API for a function with three inputs (`x,y,z`) and two outputs (`foo,bar`)

```python
def some_func(x:{'units':'m'}=np.zeros(4),
              y:{'units':'m'}=np.ones(4),
              z:{'units':None}=3) 
              -> [('foo', {'units':1/m, 'shape':4}), 
                  ('bar', {'units':'m', 'shape':4})]:

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar
```


### No direct OpenMDAO Dependency 

The use of simple Python data structures is intentional.
It allows users to build capability that has no direct dependence on OpenMDAO. 
The annotations contain all the metadata, except for the default value which can be provided by normal python syntax for default argument values. 
An overarching goal of this API is to include all the critical data in the function annotations themselves. 
No additional data should be needed when creating a `FuncComponent` other than the function itself. 

### Why return annodations must provide output names 

While the names of the inputs are guaranteed to be able to be introspected from the function definition, the same is not true for return values. 
Consider a function like this: 

```python
def ambiguous_return_func(x,y,z):
    return 3*x, 2*y+Z

```
There is no way to infer output names from that because the computation doesn't declaring intermediate variables with names at all. 
Hence out variable names have to be given as part of the function annotation. 

### Return annotations must be either list of tuple or dict object, which preserves insertion order as an implementation detail as of CPython 3.6 and as a language feature as of python 3.7

It API provides output annotations in a strictly ordered data structure so that the metadata can be matched with the correct return value. 
So return annotations must be either a list of (<var_name>, <var_meta>) or alternatively users can provide a dict
```python
def some_func(x:{'units':'m'}=np.zeros(4),
              y:{'units':'m'}=np.ones(4),
              z:{'units':None}=3) 
              -> dict(foo={'units':1/m, 'shape':4},
                      bar={'units':'m', 'shape':4}):

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar
```


### Shorthand for uniform metadata 
There is a simple case where all the the metadata for every input and output variable is the same (i.e. same size, units, value). 
In these cases, we can offer a more compact syntax with a function decorator: 

```python
@om.func_meta(units'm', shape=4, out_names=('foo', 'bar', 'baz'))
def uniform_meta_func(x, y, z): 

    foo = x+y+z
    bar = 2*x+y
    baz = 42*foo
    return foo, bar, baz
```

Any annotations provided in the function definition will take precedence over the ones given in the decorator

### Naming I/O with non-valid Python variable names

OpenMDAO keeps track of model variable names using strings, which gives a lot more flexibility. 
Users can include special characters (e.g. ":","-") that are invalid to include in Python variable names. 
Using the above API based on function introspection, the output names are still given as strings but the input names must use valid python variable syntax. 
Sometimes, this restriction can be limiting --- especially when you are adding a component to a larger model that has existing variable naming conventions you wish to follow. 
While you could work around the limitation using aliasing of the promoted names, 
it may be more convenient to provide a string based name for inputs as part of the annotation. 

```python
def some_func(x:{'units':'m', 'name'='flow:x'}=np.zeros(4),
              y:{'units':'m', 'name'='flow:y'}=np.ones(4),
              z:{'units':None, 'name'='flow:z'}=3) 
              -> [('foo',{'units':1/m, 'shape':4}), ('bar',{'units':'m', 'shape':4})] : 

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar
```
When `name` metadata is given, OpenMDAO will use the string provided instead of the argument name. 

### Variable sizing 

One unique aspect of OpenMDAO variable metadata syntax is that you can specify a scalar default value and a non-scalar size, 
and OpenMDAO interprets that to mean `np.ones(shape)*default_val`. 
For consistency, the functional API will respect the convention. 
If a shape is given as metadata, then the default value will be broadcast out to that shape. 

```python
def some_func(x:{'units':'m', 'shape':4}=0.,
              y:{'units':'m', 'shape':4}=1.,
              z:{'units':None}=3) 
              -> [('foo',{'units':1/m, 'shape':4}), ('bar',{'units':'m', 'shape':4})] : 

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar
```
Note: If `shape` metadata is given along with a non-scalar default value for the argument, then an error will be raised during setup by OpenMDAO. 
Question: how do you indicate shape as an option, or function of options, like `num_nodes`? I would really hate to use a string, maybe we could do something with a context manager to provide a name, like:

```python
with om.func.options(num_nodes=int) as num_nodes:
    def some_func(x:{'units':'m', 'shape':num_nodes}=0.,
                  y:{'units':'m', 'shape':num_nodes}=1.,
                  z:{'units':None}=3)
                  -> [('foo',{'units':1/m, 'shape':4}), ('bar',{'units':'m', 'shape':num_nodes})]:

        foo = np.log(z)/(3*x+2*y)
        bar = 2*x+y
        return foo, bar
```
You could pass in multiple name, option metadata pairs and the __enter__() must return a tuple-like that could be unpacked as expected.
```python
with om.func.options(N_cp=int, num_nodes=...) as N_cp, num_nodes:
    ...
```
The arguments could also be a dict and provide meta data (like default or allowed options). An optional syntax would be to treat a value as a default. Both syntax options are shown:
```python
with om.func.options(
N_cp=0, # implies default value of 0, of type type(0)
special_moode=dict(default="opt1",values={"opt1","opt2","opt3"},desc=""), # can assign kwarg dict gets passed to  component.options.declare(**kwargs)
) as N_cp, num_nodes:
    ...
```


## Adding a FuncComponent to a model 
OpenMDAO will add a new Component to the standard library called `FuncComponent`, which will accept one or more functions as arguments and will then create the necessary component and all associated I/O

```python
def some_func(x:{'units':'m'}=np.zeros(4),
              y:{'units':'m'}=np.ones(4),
              z:{'units':None}=3) 
              ->  [('foo',{'units':1/m, 'shape':4}), ('bar',{'units':'m', 'shape':4})] : 

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar

@OMmeta(units'm', shape=4, out_names=('baz'))
def some_other_func(x, y): : 
    return x**y

comp = om.ExplicitFuncComp(some_func, some_other_func)    
```

The resulting `comp` component instance would have three inputs: `x`, `y`, `z`. 
It would have three outputs `foo`, `bar`, `baz`. 
Note that no two output names on different functions can be the same, since that would cause a name conflict in the output list. 


NOTE: I would actually discourage combining multiple functions into an single component; I think it would invite name conflicts and breaks options syntax. I think a preferred signature would be `ExplicitFuncComp(func_to_comp, **comp_options_kwargs)` and naming convention should be `<function name>_comp`. So for an earlier version of `some_func` with the `num_nodes` option declared, we would do `some_func_comp = om.ExplicitFuncComp(some_fnc, num_nodes=3)` or equivalent.

## Full API Access for Components that might be called in setup

In a Component's `setup` method, the component is configured by calling methods like `self.add_input` and `self.add_output`. These are provided by the annotations discussed above. Other methods are sometimes called like `declare_partials`, which will be provided by decorators in the `om.func` namespace.


### Using `declare_partials` and `declare_coloring`

For example:

```python
@om.func.declare_coloring(wrt='*', method='cs')
@om.func.declare_partials('*', '*', method='cs')
def some_func(x:{'units':'m'}=np.zeros(4),
              y:{'units':'m'}=np.ones(4),
              z:{'units':None}=3) 
              -> dict(foo={'units':1/m, 'shape':4},
                      bar={'units':'m', 'shape':4}):

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar

some_func_comp = om.ExplicitFuncComp(some_func)    
```

The API would exactly mimic current ExplicitComponent and ImplicitComponent. 
The OpenMDAO API respects the partials declaration order, with later calls taking precedence over earlier ones. 
Care may need to be taken to ensure ordering is consistent. I am showing examples in forward order, but I believe the decorators would essentially process in reverse order

Also note that the local reference to `some_func` should preserve its original `__call__` so that it can be used directly with normal python function call syntax.

### Providing a `compute_partials` or `compute_jacvec_product`

Users can provide a secondary function that gives `compute_partials` functionality. 
For `compute_partials`, the argument structure must follow that of the primary function, with the last argument being a provided Jacobian object. 
Just like a normal OpenMDAO component, the shape of the expected derivative data is determined by the shapes of the inputs and outputs and whether or not any rows and cols are given. 

```python

def J_some_func(x, y, z, J): 

    J['foo', 'x'] = -3*np.log(z)(3*x+2*y)**2 
    J['foo', 'y'] = -2*np.log(z)(3*x+2*y)**2 
    J['foo', 'z'] = 1/(3*x+2*y) * 1/z

    J['bar', 'x'][:] = 2 # need to set all elements of array
    J['bar', 'y'][:] = 1 # need to set all elements of array

@om.func.declare_partials(of='foo', wrt='*', rows=np.arange(4), cols=np.arange(4))
@om.func.declare_partials(of='bar', wrt=('x', 'y'), rows=np.arange(4), cols=np.arange(4))
@om.func.compute_partials(J_some_func)
def some_func(x:{'units':'m'}=np.zeros(4),
              y:{'units':'m'}=np.ones(4),
              z:{'units':None}=3)
              -> dict(foo={'units':1/m, 'shape':4},
                      bar={'units':'m', 'shape':4}):
    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar

some_func_comp = om.ExplicitFuncComp(some_func,)    
```

Note, the declaration of `J_some_func` as well as the (decorated) declaration of `some_func` could be under the `om.func.options` context manager if needed.
Just like a normal explicit component, if you are using the matrix free API then you should not declare any partials. 
The matrix vector product method method signature will expect three additional arguments added beyond those in the nonlinear function: `d_inputs, d_outputs, mode` 

```python

def jac_vec_some_func(x, y, z, d_inputs, d_outputs, mode):
    ...  

@om.func.compute_jcavec_product(jac_vec_some_func)
def some_func(x:{'units':'m'}=np.zeros(4),
              y:{'units':'m'}=np.ones(4),
              z:{'units':None}=3) 
               -> dict(foo={'units':1/m, 'shape':4},
                      bar={'units':'m', 'shape':4}):
                  ('compute_jacvec_product', jac_vec_some_func), 
                 ] : 

    foo = np.log(z)/(3*x+2*y)
    bar = 2*x+y
    return foo, bar

some_func_comp = om.ExplicitFuncComp(some_func,)    
```


## Implicit API Description

Note: I want to look at this more, but it seems to me that since ImplicitComponents require so many additional methods, I would rather use the class syntax; I think supporting all of the declarations that would be necessary to construct typical ImplicitComponents is a lot of work without much payoff. If we can use some of the syntax sugar above to reduce boilerplate for the `initialize` and `setup` method definitions, it may be worth looking into that.

Implicit components must have at least an `apply_nonlinear` method to compute the residual given values for input variables and implicit output variables (a.k.a state variables). 
From the perspective of the residual computation, both input *variables* and implicit output *variables* are effectively input *arguments*. 
This creates a slight API challenge, because it is ambiguous which arguments correspond to input or output variables. 

For explicit components, output variable names were given as part of the metadata in the function return annotation. 
That approach is also used for implicit components with one slight change to accommodate the output-variable function arguments. 
Output names must still be given in the return metadata, but they must name-match one of the function arguments. 

```python

@om.func_meta(units=None, shape=1)
def some_implicit_resid(x, y) -> [('y', None),]:

    R_y = y - tan(y**x)
    return R_y

comp = om.ImplicitFuncComp(some_implicit_resid,)    
```

If you want to use OpenMDAO variables names that contain characters that are non valid for arguments, then provide `name` metadata for that output. 

```python

@om.func_meta(units=None, shape=1)
def some_implicit_resid(x, y)->[('y',{'name':'foo:y'})]:

    R_y = y - tan(y**x)
    return R_y

comp = om.ImplicitFuncComp(some_implicit_resid,)    
```


A `solve_nonlinear` method can also be specified as part of the metadata: 

```python
@om.func_meta(units=None, shape=1, out_names=['R_x', 'R_y'])

def some_implict_solve(x,y)

@om.func.set_solve_nonlinear(some_implicit_solve)
def some_implicit_resid(x, y)-> dict(y={'name':'foo:y'}): 
    R_x = x + np.sin(x+y)
    R_y = y - tan(y)**x
    return R_x, R_y

comp = om.ImplicitFuncComp(some_implicit_resid,)    
```

### Providing a `linearize` and/or `apply_linear` for implicit functions

The derivative APIs look very similar to the ones for those of the explicit functions, but with different method names to match the OpenMDAO implicit API. 
Implicit components use `linearize` and `apply_linear` methods (instead of the analogous `compute_partials` and `compute_jacvec_product` methods). 

```python

def deriv_implicit_resid(x, y, J): 
    ... 

@om.func_meta(units=None, shape=1)
def some_implicit_resid(x, y)->[('y',{'name':'foo:y'}), 
                                ('linearize', deriv_implicit_resid)]:

    R_y = y - tan(y**x)
    return R_y

comp = om.ImplicitFuncComp(some_implicit_resid,)    
```

## Helper decorators

Though the annotation API is designed to be usable without any OpenMDAO dependency, the dictionary and list based syntax may be somewhat cumbersome. 
OpenMDAO can provide some decorators to make the syntax slightly cleaner. 

One example is the `func_meta` decorator already described. 
Two more decorators, `in_var_meta` and `out_var_meta`, will be provided to specify metadata for individual variables. 
These decorators can be stacked to fully defined the component and variable metadata.

```python

def deriv_implicit_resid(x, y, J): 
    ... 

@om.in_var_meta('x', units=None, shape=1)
@om.out_var_meta('y', units=None, shape=1, name='foo:y')
@om.func_meta(linearize=deriv_implicit_resid)
def some_implicit_resid(x, y):

    R_y = y - tan(y**x)
    return R_y

comp = om.ImplicitFuncComp(some_implicit_resid,)    
```
Or what if in_var doesn't need the name and just takes it from inspection? 
