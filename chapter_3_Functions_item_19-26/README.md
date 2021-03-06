# Chapter 3. Functions

## Item 19: Never Unpack More Than Three Variables When Function Return Multiple Values

* Unpacking allows seemingly return more than one value:
```python
def get_stats(numbers):
    minimum = min(numbers)
    maximum = max(numbers)
    return minimum, maximum

lengths = [63, 73, 72, 60, 67, 66, 71, 61, 72, 70]

minimum, maximum = get_stats(lengths)

print(f"Min: {minimum}, Max: {maximum}")
```
    >>>
    Min: 60, Max: 73

* Multiple values also can be packed using `*stared_expressions`:
```python
def get_avg_ratio(numbers):
    average = sum(numbers) / len(numbers)
    scaled = [x / average for x in numbers]
    scaled.sort(reverse=True)
    return scaled

longest, *middle, shortest = get_avg_ratio(lengths)

print(f"Longest: {longest:>4.0%}")
print(f"Shortest: {shortest:>4.0%}")
```
    >>>
    Longest: 108%
    Shortest: 89%

* Now, imagine you need to show also min, max, avg, median and total population. It can be dome by extending our previous function:
```python
def get_stats(numbers):
    maximum = max(numbers)
    minimum = min(numbers)
    count = len(numbers)
    average = sum(numbers) / count

    sorted_numbers = sorted(numbers)
    middle = count // 2
    if middle % 2 == 0:
        lower = sorted_numbers[middle - 1]
        upper = sorted_numbers[middle]
        median = (lower + upper) / 2
    else:
        median = sorted_numbers[middle]
    return minimum, maximum, average, median, count

minimum, maximum, average, median, count = get_stat(numbers)

print(f"Min: {minimum}, Max: {maximum}")
print(f"Average: {average}, Median: {median}, Count {count}")
```
    >>>
    Min: 60, Max: 73
    Average: 67.5, Median: 70, Count 10

But, this approach has two main problems. First, all the return values are numeric and it is easy to reorder them accidentally and not to notice that. Which, will lead to the hard-to-find bugs in the future. Second, is that line of unpacking of the values is to long and, according to the PEP 8, it should be divided into more lines. Which will hurt readability. 

To avoid these problems you should never use more than tree(3) variables when unpacking multiple return values from a function. Instead, small class or `namedtuple` maybe used.

## Item 20: Prefer Raising Error to Returning `None`
Sometimes returning `None` in helper function seems natural. Like in case of division function:
```python
def careful_divide(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        return None
```
It can work properly:
```python
x, y = 1, 0
result = careful_divide(x, y)
if result is None:
    print("Invalid inputs")
```
But, `None` could also mean `False`. So, following code will be wrong:
```python
x, y = 0, 5

result = careful_divider(x, y)
if not result:
    print("Invalid inputs") # This runs, but shouldn't.
```

* To avoid such problems, you may return tuple with two values indicating of division was successful and the result of the division:
```python
def careful_divider(a, b):
    try:
        return True, a / b
    except ZeroDivisionError:
        return False, None

success, result = careful_division(x, y)
if not success:
    print("Invalid inputs")
```
But it alo can lead to the same problems, for instance, `_` underscore variable name may be used to ignore part or the `tuple`:
```python
_, result = careful_division(x, y)
if not result:
    print("Invalid inputs")
```

* The next better way is to raise an exception and make the caller to handle it:
```python
def careful_division(a, b):
    try:
        return a / b
    except ZeroDivisionError as e:
        raise ValueError("Invalid inputs")
```
This way the caller do not need to check if the result is valid and use it in `else` block:
```python
x, y = 3, 9
try:
    result = careful_division(x, y)
except ValueError:
    print("Invalid inputs")
else:
    print(f"Result is {result:>.1}")
```

* This approach can be extended by using `type annotation` and `docstring`:
```python 
def careful_division(a: float, b:float) -> float:
    """Divides a by b.

    Raises:
        ValueError: When inputs cannot be divided.
    """
    try:
        return a / b
    except ZeroDivisionError as e:
        raise ValueError("Invalid inputs")
```

## Item 21: Know How Closures Interact with Variable Scope
Let's imagine that you want to sort `list` of numbers, but prioritize one group of number to come first. You can do it by passing a helper function as a `key` argument to the `sort()` method.  
```python
def sort_priority(values, group):
    def helper(x):
        if x in group:
            return (0, x)
        return (1, x)
    values.sort(key=helper)
```
This function works for simple inputs:
```python
numbers = [8, 3, 1, 2, 5, 4, 7, 6]
group = {2, 3, 5, 7}
sort_priority(numbers, group)
print(numbers)
```
    >>>
    [2, 3, 5, 7, 1, 4, 6, 8]

There are 3 reasons why this is working:
1. Python supports *Closures* - that is, functions that refer to the variable from scope in which they were defined. That is why `helper` function can access `group` argument from `sort_priority`.
2. Functions are *first-class* objects in Python. They you can refer them directly, assign the to the variables, pass them as a arguments to other functions, compare them in expressions and if statements, and so on. This is why `sort()` method can accept closure `helper` function as a `key` argument. 
3. Python has specific rules for comparing elements. Fist, it compares element with index `0`, then, if they are equal, compare elements and index `1` and so on. That is why the return values from the `helper` closure causes the sort order to have two distinct groups. 


It'd be nice if this function also return a flag if the numbers from group is present. Adding this seems like not a hard problem:
```python
def sort_priority2(values, group):
    found = False
    def helper(x):
        if x in group:
            found = True
            return (0, x)
        return (1, x)
    values.sort(key=helper)
    return found

found = sort_priority2(numbers, group)
print('Found:', found)
print(numbers)
```
    >>>
    Found: False
    [2, 3, 5, 7, 1, 4, 6, 8]

Function is not working. This happens because variable assignment does not go outside of the helper's scope. 

    Python scans for references in following order"

    1. The current function's scope.
    2. Any enclosing scopes (such as other containing functions).
    3. The scope of the module that contains the code (aka *Global scope*).
    4. The built-in scope (that contains functions line `len()` and `str()`).

    If it cant find it in this places, `NameError` exception will be raised.

* To get date out of a closure, `nonlocal` syntax can be used:
```python
def sort_priority3(values, group):
    found = False
    def helper(x):
        nonlocal found
        if x in group:
            found = True
            return (0, x)
        return (1, x)
    values.sort(key=helper)
    return found


found = sort_priority3(numbers, group)
print('Found:', found)
print(numbers)
```
    >>>
    True
    [2, 3, 5, 7, 1, 4, 6, 8]

However, you should be careful with `global` and `nonlocal` statements and use them in relatively small functions only. 

If it is getting complicated, better to use wrapper class:
```python
class Sorter:
    def __init__(self, group):
        self.group = group
        self.found = False
    def __call__(self, x):
        if x in self.group:
            self.found = True
            return (0, x)
        return (1, x)

sorter = Sorter(group)
numbers.sort(key=sorter)
assert sorter.found is True
```
## Item 22: Reduce Visual Noise with Variable Positional Argument
Positional arguments in functions may reduce visual noise and increase readability. Positional arguments are often called `varargs` or `star args` (because of the way they are declared `*args`).

* With fixed number of parameters you can have a function like this:
```python
def log(message, values):
    if not values:
        print(message)
    else:
        values_str = ", ".join(str(x) for x in values)
        print(f"{message}: {values_str}")

log("My numbers are", [1, 2])
log("Hi there", [])
```
    >>>
    My numbers are: 1, 2
    Hi there

As you can see, call with empty list does not look great. You can avoid it with prefixing optional arguments with `*`:
```python
def log(message, *values): # The only change
    if not values:
        print(message)
    else:
        values_str = ", ".join(str(x) for x in values)
        print(f"{message}: {values_str}") 

log("My numbers are", [1, 2])
log("Hi there") # And a function call is changed.
```
    >>>
    My numbers are: 1, 2
    Hi there

* A sequence can be passed to the function call using `*` operator:
```python
favorites = [7, 33, 99]
log('Favorite colors', *favorites)
```
    >>>
    Favorite colors: 7, 33, 99

* There are two problems with accepting variable number of positional arguments. 
1. Optional positional arguments are always turned into `tuple`. This mean if a generator function is called with `*` argument it will be iterated until its over, which can eat a lot of memory and result in crush:
```python
def my_generator():
    for i in range(10**100):
        yield i

def my_func(*args):
    print(args)

it = my_generator()
my_func(*it)
```
`*args` parameters are good in situation when you now the number of arguments and it is a relatively small number. 

2. Second problem is: you cannot add new positional arguments in the future without breaking old calls (backward incompatible):
```python
def log(sequence, message, *values):
    if not values:
        print(f"{sequence} - {message}")
    else:
        values_str = ", ".join(str(x) for x in values)
        print(f"{sequence} - {message}: {values_str}")

log(1, 'Favorites', 7, 33)          # RIGHT New call
log(1, 'Hi there')                  # RIGHT New call
log('Favorite numbers', 7, 33)      # WRONG Old usage breaks
```
    >>>
    1 - Favorites: 7, 33
    1 - Hi there
    Favorite numbers - 7: 33

This kind of bugs are hurd to track down because function runs without exception. To avoid this, you can add new functionality by using keyword-only arguments or, even more robust, use type annotation. 

## item 23: Provide Optional Behavior with Keyword Arguments
In Python argument to a function call maybe positional or keyword. You may use keyword-only arguments in any order in the call:
```python
def remainder(number, divisor):
    return number % divisor

remainder(20, 7)
remainder(20, divisor=7)
remainder(number=20, divisor=7)
remainder(divisor=7, number=20)
```

* Positional arguments should be passed before keyword arguments:
```python
remainder(number=20, 7)
```
    >>>
    Traceback ...
    SyntaxError: positional argument follows keyword argument

* And, each argument can be specified only once:
```python
remainder(20, number=7)
```
    >>>
    Traceback ...
    TypeError: remainder() got multiple values for argument 'number'

* If you have a `dict` and want to have it content to pass as function call - you can use `**` and keys of the `dict` will be used as the keyword argument names and values as a arguments:
```python
my_kwargs = {
    'number': 20,
    'divisor': 7,
}
remainder(**my_kwargs)
```
    >>>
    6
* This cool feature can be used with two(2) dictionaries if they don't have same keys:
```python
my_kwargs = {
    "number": 20,
}
other_kwargs = {
    "divisor": 7,
}
reminder(**my_kwargs, ^^other_kwargs)
```
    >>>
    6

* If you want function to catch all the named arguments, you can use `**kwargs` as well:
```python
def print_parameters(**kwargs):
    for key, value in kwargs.items():
        print(f"{key} = {value}")

print_parameters(alpha=1.5, beta=9, gamma=4)
```
    >>>
    alpha = 1.5
    beta = 9
    gamma = 4

Keyword arguments provides three significant benefits:
1. First benefit is that it is always clearer to the new caller of the function which argument is which, without locking in to the implementation.
2. Second is that you can assign default values to the argument, thus provide additional functionality. 
    For instance, let's look at the function computing the rate of the fluid going into the vat:

```python
def flow_rate(weight_diff, time_diff):
    return weight_diff / time_diff

weight_diff = 0.5
time_diff = 3

flow = flow_rate(weight_diff, time_diff)
print(f"{flow:.3}, kg per second")
``` 
    >>>
    0.167 kg per second

Now, let's say you need to add a functionality to change a period from second to hours or more. We can do it by adding new `period` argument. But we dint want to specified it each call. We want the function to compute rate at per second by default and only, if needed, change the period. We can achieve this by using `default` value for the `argument`:
```python
def flow_rate(weight_diff, time_diff, period=1):
    return (weight_diff / time_diff) * period


flow_per_second = flow_rate(weight_diff, time_diff)
flow_per_hour = flow_rate(weight_diff, time_diff, period=3600)

print(f"{flow_per_second:.3} kg per second")
print(f"{flow_per_hour} kg per hour")
```
    >>>
    0.167 kg per second
    600.0 kg per hour

3. Third reason is that keyword arguments provide you with the way to add functionality without breaking backward compatibility. This mean you can add functionality without big changes in existing code base. 

Here, we will add new functionality `units_per_kg` to the function:
```python
def flow_rate(weight_diff, time_diff, period=1, units_per_kg=1):
    return ((weight_diff * units_per_kg)/ time_diff) * period

pounds_per_hour = flow_rate(weight_diff, time_diff, period=3600, units_per_kg=2.2)
print(f"{pounds_per_hour} pounds per hour")
```
    >>>
    1320.0 pounds per hour

The only problem with this approach is that keyword argument can also be called using its position. Following call will work:
```python
pounds_per_hour = flow_rate(weight_diff, time_diff, 3600, 2.2)
```


## Item 24: Use `None` and Docstrings to Specify Dynamic Default Argument
Sometimes you need to make a non-static type the default value. For instance, like in logger function, you want to print time of event occurrence:
```python
from time import sleep
from datetime import datetime


def log(message, when=datetime.now()):
    print(f"{when}: {message}")

log("Hi there!")
sleep(0.1)
log("Hello again!")
sleep(0.1)
log("Last time")
```
    >>>
    2020-01-06 08:44:39.286206: Hi there!
    2020-01-06 08:44:39.286206: Hello again!

This does not work. Because `now()` show the time when function is defined(which is, usually, when module is loaded).

* Common way to achieve this is to use `None` default value and explain the behavior docstring:
```python:
def log(message, when=None):
    """Log a message with a timestamp.
    
    Args:
        message: Message to print.
        when: datetime of when the message occurred.
            Defaults to the present time.

    """
    if when is None:
        when = datetime.now()
    print(f"{when}: {message}")
```
    >>>
    2020-01-06 09:00:14.731529: Hi there!
    2020-01-06 09:00:14.832437: Hello again!

* It is especially important when the arguments are mutable. Let's say we want to load a value in JSON format and if the decoding fails, tp return an empty `dict`:
```python
import json


def decode(data, default={}):
    try:
        return json.loads(data)
    except ValueError:
        return default
```
The problem is that default `dict` will be shared with all the call of the function. Because it will be defined only once at module loading. Which will have a surprising behavior:
```python
foo = decode("bad date")
foo["stuff"] = 5
bar = decode("also bad")
bar["meep"] = 1

print("Foo:", foo)
print("Bar:", bar)
```
We were expecting two different dictionaries but change in first is affecting the other. It is because this is one `dict` the `default` one:
```python
assert foo is bar
```
Not to have this problem we should default to `None` and explain behavior in docstring:
```python 
def decode(data, default=None):
    """Load JSON data from a string.
    Args:
        data: JSON data to decode.
        default: Value to return if decoding fails.
            Defaults to an empty dictionary.
    """
    try:
        return json.loads(data)
    except ValueError:
        if default is None:
            default = {}
    return default


foo = decode("bad date")
foo["stuff"] = 5
bar = decode("also bad")
bar["meep"] = 1

print("Foo:", foo)
print("Bar:", bar)

```
    >>>
    Foo: {'stuff': 5}
    Bar: {'meep': 1}


This approach works well with type annotations:
```python
from typing import Optional


def log_typed(message: str, 
            when: Optional[datetime]=None) -> None:
    """Log a message with a timestamp.
    Args:
        message: Message to print.
        when: datetime of when the message occurred.
            Defaults to the present time.
    """
    if when is None:
        when = datetime.now()
    print(f'{when}: {message}')
```

## Item 25: Enforce Clarity with Keyword-Only and Positional-Only Arguments 
Following modification of the division function now can ignore division by zero error by returning infinity and will return `0` for overflow. 
```python
def safe_division(number, divisor, ignore_overflow, ignore_zero_division):
    try:
        return number / divisor
    except OverflowError:
        if ignore_overflow:
            return 0
        else:
            raise
    except ZeroDivisionError:
        if ignore_zero_division:
            return float("inf")
        else:
            raise

result = safe_division(1.0, 10**500, True, False)
print(result)

result = safe_division(1.0, 0, False, True)
print(result)
```
    >>>
    0
    inf

* It is very easy to confuse the positions of two last booleans. One way to make the function call less error prone is add default values to the Boolean arguments:
```python
def safe_division_b(number, divisor, ignore_overflow=False, ignore_zero_division=False)
    ...
```
* Now, specific behavior can be called by overriding default values:
```python
result = safe_division_b(1.0, 10**500, ignore_overflow=True)
print(result)
result = safe_division_b(1.0, 0, ignore_zero_division=True)
print(result)
```
    >>>
    0
    inf

* However, this boolean arguments still can be called using their position:
```python
assert safe_division_b(1.0, 10**500, True, False) == 0
```

For complex functions it is recommended to use keyword-only arguments. It is achieved by separating arguments with `*`:
```python
def safe_division_c(number, divisor, *, ignore_overflow=False, ignore_zero_division=False)
    ...
```

* Now, following function call won't work:
```python
safe_division(1.0, 10**500, True, False)
```
    >>>
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>   
    TypeError: safe_division() takes 2 positional arguments but 4 were given    

* However, the problem with first two arguments still exist, when you can mix position with keyword:
```python
assert safe_division_c(number=2, divisor=5) == 0.4
assert safe_division_c(divisor=5, number=2) == 0.4
assert safe_division_c(2, divisor=5) == 0.4
```

* Let's say, that later we have decided to change the names for some random reason. Now previous calls to the function will break: 
```python
def safe_division_c(numerator, denominator, *, # Changed
                    ignore_overflow=False,
                    ignore_zero_division=False):
    ...

safe_division_c(number=2, divisor=5)
```
    >>>
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: safe_division_c() got an unexpected keyword argument 'number'

This is especially problematic because the name of the arguments never been intended to be the part of the interface. They are just convenient variable names.

* Luckily in Python 3.8 there is a way to specifically make *positional-only arguments*:
```python
def safe_division_d(numerator, denominator, /, *, # Changed
                    ignore_overflow=False,
                    ignore_zero_division=False):

    ...
print(safe_division_d(2, 5))
print(safe_division_d(numerator=2, denominator=5))
```
    >>>
    0.4
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: safe_division_d() got some positional-only arguments passed as keyword arguments: 'numerator, denominator'

* Notable consequence of this new features is: any parameter placed between `/` and `*` can be called either by position or keyword (which is default for python functions):
```python
def safe_division_e(numerator, denominator, /, 
                    ndigits=10, *, # Changed
                    ignore_overflow=False,
                    ignore_zero_division=False):
    try:
        fraction = numerator / denominator
        return round(fraction, ndigits)
    except OverflowError:
        if ignore_overflow:
            return 0
        else:
            raise
    except ZeroDivisionError:
        if ignore_zero_division:
            return float("int")
        else:
            raise


result = safe_division_e(22, 7)
print(result)

result = safe_division_e(22, 7, 5)
print(result)

result = safe_division_e(22, 7, ndigits=2)
print(result)
```
    >>>
    3.1428571429
    3.14286
    3.14

## Item 26: Define Function Decorators with `functools.wraps`
Python has specific `Decorators` that can be applied to the functions. This decorators can interact with a function before and after of each function call. Therefor, `decorators` can manipulate with values passed to the function arguments and return values of the functions. These decorators are used for wide variate of cases. This functionality can be used in debugging, enforcing semantics, registering functions. 

For example, we want to have a log function, which will print arguments and corresponding return values. This can be helpful in debugging nested function calls. Let's define our decorator function using `*args` and `**kwargs`:
```python
def trace(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print(f"{func.__name__}({args!r}, {kwargs!r}) "
                f"-> {result!r}")
        return result
    return wrapper
```
Decorators can be applied to the function using `@` symbol:
```python
@trace
def fibonacci(n):
    """Returns n-th Fibonacci number"""
    if n in (0, 1):
        return n
    return (fibonacci(n - 2) + fibonacci(n - 1))
```
It is equivalent to the: 
```python
fibonacci = trace(fibonacci)
```
The decorated function run wrapper function code before and after fibonacci runs. 
```python
fibonacci(4)
```
    >>>
    fibonacci((0,), {}) -> 0
    fibonacci((1,), {}) -> 1
    fibonacci((2,), {}) -> 1
    fibonacci((1,), {}) -> 1
    fibonacci((0,), {}) -> 0
    fibonacci((1,), {}) -> 1
    fibonacci((2,), {}) -> 1
    fibonacci((3,), {}) -> 2
    fibonacci((4,), {}) -> 3

The problem is that value returned by this function, doesn't think its called fibonacci:
```python
print(fibonacci)
```
    >>>
    <function trace.<locals>.wrapper at 0x7f0103ccfee0>

Now, debugger won't work and things like `help()` won't work either. We would expect that `help()` function will return our doc string `"""Returns n-th Fibonacci number"""`, but it doesn't:
```python
help(fibonacci)
```
    >>>
    Help on function wrapper in module __main__:

    wrapper(*args, **kwargs)

Also, object serializers break because they cannot determine the location of the original function that been decorated:
```python
import pickle 


pickle.dumps(fibonacci)
```
    >>>
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: Can't pickle local object 'trace.<locals>.wrapper'

The solution is to use `wraps` decorator from `functools` build-in. This is decorator to write decorators. When you use it, it copies all the inner information of the function, and makes it available to the outer function:
```python
from functools import wraps


def trace(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print(f"{func.__name__}({args!r}, {kwargs!r}) "
                f"-> {result!r}")
        return result
    return wrapper
```
Now `help()` and `pickle.dumps()` will work correctly. Moreover, functions in Python have many other important attributes, which will be available with help of the `@wraps` decorator.  

#
* [Back to repo](https://github.com/almazkun/effective_python#effective_python)