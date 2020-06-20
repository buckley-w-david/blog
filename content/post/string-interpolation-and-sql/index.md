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

`interpolate.py` contains two functions, one horrible, the other even worse.

## `parameterize_interpolated_querystring`

`paramaterize_interpolated_querystring` takes as its input a string with the same syntax as an f-string, just without the `f` prefix.

So for example, if you wanted to do this:

```
id = 5
# This is vulnerable to an SQL injection attack
cursor.execute(f"SELECT * FROM users WHERE id = {id}") 
```

You could instead do this
```
import interpolate
id = 5
f = interpolate.paramaterize_interpolated_querystring
cursor.execute(*f("SELECT * FROM users WHERE id = {id}")) 
```

Which will auto-magically be transformed into

```
cursor.execute("SELECT * FROM users WHERE id = ?", [5])
```

### How does it work?

Most of the magic is in these lines

```
frame = inspect.currentframe()
outer_frame = inspect.getouterframes(frame)[1]
possible_query_values = {**globals(), **outer_frame.frame.f_locals}
```

This takes advantage of the `inspect` module to reach up into the callers stack frame and retrieve their local variables. This is what allows us to return the list of parameter values using the variable names within the interpolation.

Taking advantage of the `ast` module then allows us to do the f-string parsing (The same way that python itself does it), and replace all instances of interpolation with a placeholder (`?` by default, but the function accepts this as an argument), resulting in the parameterized query string.

## `parameterize_interpolated_querystring_spicy`

This function is `parameterize_interpolated_querystring` cursed twin. Generally it does the same thing, except it is more powerful and far more dangerous.

You may have noticed while reading about `parameterize_interpolated_querystring` that the method I described will actually only work for interpolation like `f"value: {x}"` and fail when trying to do something like `f"value: {x+1}"`. 

This is probably fine, it seems somewhat unnecessary and dangerous to allow arbitrary expressions.

However it was a hole in functionality, and `parameterize_interpolated_querystring_spicy` is the solution to fill it.

Generally the approach is similar, use `inspect` and `ast` to get what values you need and return a parameterized version of the query. The difference is that instead of just using whatever was in the interpolation to look up the variables value, we use the `compile` and `exec` build-in functions along with some more ast manipulation to actually evaluate whatever was in there.

The gist of how that's done is here:

```
...
tree = ast.parse(f"f'{query}'")
values = tree.body[0].value.values
temp_name = '__paramaterize_interpolated_querystring_spicy_temp'
assign = ast.parse(f'{temp_name} = 0')

for node in values:
	...
		# This may be the most cursed code I have ever written
		assign.body[0].value = node.value
		assign = ast.fix_missing_locations(assign)
		exec(compile(assign, '<string>', 'exec'), globals(), outer_frame.frame.f_locals)

		query_values.append(outer_frame.frame.f_locals[temp_name])
	...
```

Using the `ast` module I build an assignment statement for the value to interpolate, `compile` and `exec` it, and then retrieve it from the outer stack frames locals.

I would have much preferred not having to pollute the outer stack frame with a variable, but I could not figure out any other technique to accomplish the same feat and retrieve the value. This could very well just be my ignorance.

The upside though is that it does work!

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

Please don't actually use any of this code.
