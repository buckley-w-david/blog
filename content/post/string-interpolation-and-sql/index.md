---
title: "String Interpolation and SQL"
date: 2020-06-19T19:52:43-04:00
draft: false
---

## Background

<div class="disclaimer">
<p>Don't use any code I provide or link to in production without giving it a serious think.</p>
</div>

SQL injection sucks, right? I'm not going to talk about it too much because firstly you already know, and secondly you can go somewhere else for a much better explanation.

```bad
# This is vulnerable to an SQL injection attack
cursor.execute(f"SELECT * FROM users WHERE id = {id}") 
```

```good
# This is not, but it is much less natural
cursor.execute("SELECT * FROM users WHERE id = ?", id)
```

What I **am** going to talk about though is a neat idea I had after noticing a set of functions in EF Core: `ExecuteSqlInterpolated` and `FromSqlInterpolated`.

It reminded me of some conversations that a friend [ Mike Williamson ](https://github.com/sleepycat) and I had while we were working together at the Canadian Digital Service around how SQL injection is a particularly annoying trap to fall into since the obvious way to accomplish constructing a query with dynamic pieces of data will result in vulnerability.

Mike wrote a [ blog post ](https://mikewilliamson.wordpress.com/2018/10/22/tagged-template-literals-and-the-hack-that-will-never-go-away) himself about the subject back in 2018 and how javascript has a neat way to deliver the developer experience of string interpolation without compromising security.

I encourage you to read that post before continuing on as it makes a lot of good points to understand where I'm going with this.

---

So that's .NET and Javascript that have solutions to this problem, so good for developers working with those languages. I'd bet that some other languages have their own solution to the problem, but what I'm interesting in discussing today is python.

As far as I know, when dealing with a database in python the expectation is that you will use an ORM of some sort, and if working with raw SQL will simply use prepared statements.

Taking inspiration from other languages, I wanted to be able to use the syntax of string interpolation (Specifically [f-strings](https://www.python.org/dev/peps/pep-0498/)) to construct SQL queries without the danger of injection attacks.

The only problem is unlike javascript python doesn't have the same tagged template literals, and unlike C# there is no way to supply a templated string such that a function can act on it before it is formatted into a string, making the values embedded within unreachable.

One thing that python does have however is very powerful introspection capabilities, which we can (ab)use to accomplish something very close to our goal of SQL safe string interpolation.

The [ inspect ](https://docs.python.org/3/library/inspect.html) and [ ast ](https://docs.python.org/3/library/ast.html) modules allow for lots of wacky hijinks, but the methods we will be using today are `inspect.currentframe`, `inspect.getouterframes` and `ast.parse`.

Without further ado, I introduce... [ interpolate.py ](https://github.com/buckley-w-david/parameterized-interpolated-sql-queries/blob/master/interpolate.py)!

`interpolate.py` contains a single, extremely cursed, function.

## `parameterize_interpolated_querystring`

`paramaterize_interpolated_querystring` takes as its input a string with the same syntax as an f-string, just without the `f` prefix.

So for example, if you wanted to do this:

```
id = 5
# This is vulnerable to an SQL injection attack
cursor.execute(f"SELECT * FROM users WHERE id = {id}") 
```

You could instead do this:
```
import interpolate
id = 5
f = interpolate.paramaterize_interpolated_querystring
cursor.execute(*f("SELECT * FROM users WHERE id = {id}")) 
```

Which will auto-magically be transformed into:

```
cursor.execute("SELECT * FROM users WHERE id = ?", [5])
```

### How does it work?

Most of the magic is in these lines:

```
tree = ast.fix_missing_locations(ast.parse(f'f{json.dumps(query)}'))
values = tree.body[0].value.values

frame = inspect.currentframe()
outer_frame = inspect.getouterframes(frame)[1] # element 1 is the parent stack frame

outer_locals = outer_frame.frame.f_locals.copy()
```

This takes advantage of the `inspect` module to reach up into the callers stack frame and retrieve their local variables. This is what allows us to later use these values as part of the call to resolve the interpolation. The `dict` of locals is copied instead of used directly so that we can add to it without polluting the parent namespace.

By using `ast.parse`, the string is parsed as an f-string the same way that python itself does it. `json.dumps` is used for quote escaping so that the code isn't *itself* vulnerable to an injection attack. 

<div class="disclaimer">
<p> This is the main reason you shouldn't actually do this, because while I have thought about the issue and attempted to get around it, there are no guarantees that an attacker would not be able to find a way to inject code in such a way that it's run by the python interpreter.</p>
</div>

Once all that bookkeeping is done, we can move on to building the return.

```
temp_name = '__parameterize_interpolated_querystring_temp'
assign = ast.fix_missing_locations(ast.parse(f'{temp_name} = 0'))

paramaterized_query = []
query_values = []

# An f-string has two parts
for node in values:
    # Constants, which are just sections of static strings
    if isinstance(node, _ast.Constant):
        paramaterized_query.append(node.value)
    # And FormattedValue's, that have whatever is needed to calculate the result of the interpolation
    elif isinstance(node, _ast.FormattedValue):
        paramaterized_query.append(placeholder)

        # This may be the most cursed code I have ever written
        assign.body[0].value = node.value # We pull off the calculation node and attach it to our dummy assignment
        exec(compile(assign, '<string>', 'exec'), globals(), outer_locals)

        query_values.append(outer_locals[temp_name])

return (''.join(paramaterized_query), query_values)

```

Using the `ast` module I build an assignment statement to store the result of resolving the interpolation (to the `temp_name` variable). 

Then we go through the f-string AST, and switch out the value in our assignment AST with the one from the query f-string, `compile` and `exec` it, then retrieve the value that was stored within our `outer_locals` dictionary.

This is extremely danger.

But it does work.


```
>>> import interpolate
>>> f = interpolate.parameterize_interpolated_querystring_spicy
>>> x = 1
>>> y = 2
>>> z = 3
>>> print(f('''INSERT INTO users (col1, col2, col3) VALUES ({x+1}, {y+2}, {z+3})'''))
('INSERT INTO users (col1, col2, col3) VALUES (?, ?, ?)', [2, 4, 6])
>>> def do_something(value):
...     return value*5
... 
>>> print(f('''INSERT INTO users (col1, col2, col3) VALUES ({do_something(x+7)}, {y+2}, {z+3})'''))
('INSERT INTO users (col1, col2, col3) VALUES (?, ?, ?)', [40, 4, 6])
```

## Conclusion

SQL injection sucks, but we should probably develop our tools a bit more to make it so the frictionless path is the safe one, instead of the other way around.

If you'd like to go take a look at the actual source, [it's on my GitHub](https://github.com/buckley-w-david/parameterized-interpolated-sql-queries/blob/master/interpolate.py).

Please don't actually use any of this code.
