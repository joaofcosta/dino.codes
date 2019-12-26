---
title: "Python Tuple Syntax Is Confusing"
date: 2019-12-23T12:41:57Z
description: "Post about how Python's tuple syntax can make you run around trying to fix bugs that are easily
avoided."
draft: false
truncated: false
---

Creating a list or a set with only one element, in Python, is pretty straight forward:

```python
# one element list.
["this is a list"]

# one element set.
{"this is a set"}
```

However, that's not the case if you want to create a tuple with only one element:

```python
("sorry! this is not a tuple")
```

Yup, unfortunately the way you create a one element tuple does not conform with
the rest of the data structures in Python which you can create using "special" characters.

If you want to create a tuple with only one element you must not forget the
final comma:

```python
("this is now a tuple thanks to the comma",)
```

I think this is bad. First, because it's different from all the other listed above, and second,
because a string, as well as a list, a set or a tuple, is also enumerable. So, for functions that
are expecting enumerables, if you provide them with a string they will "sort of" work.

A coworker of mine justified the behaviour with:

> "But you need the parenthesis in order to separate math operations.
> How would you differentiate between a tuple with one int and a parenthesis which is only
> wrapping a mathematical operation?"

Namely:

```python
(2)

(1 + 1)
```

Which is an understable argument seeing as we still need the ability to use paranthesis in
mathematical operations but, the truth is, when I think of the first I think we're creating a tuple,
however, its result is the same as the second expression.

## Getting bit by it!

Why did I bother making a blog post about this? Mostly because I got bitten by this little Python
behaviour at work.

I was updating one codebase's tests, which uses
[aioresponse](https://github.com/pnuckowski/aioresponses) in order to mock web requests when
working with [aiohttp](https://aiohttp.readthedocs.io/en/stable/).  We were using it as a context
manager, and you can provide a `passthrough` parameter to it, this should be an iterable, which
contains all the hosts that aioresponse must not mock. Since we were doing service level tests we
don't want to mock requests to the actual service's URL.

This is how the specific code that uses aioresponse was when I started working on it:

```python
with aioresponses(passthrough=(service_host, external_url)) as responses:
    yield responses
```

`service_host` refers to the URL of the service being updated/maintained, while `external_url` was
the URL of another service in which the one being updated depends on.

I decided to update this so as to mock the calls to the `external_url` since we want to avoid,
as much as possible, having to make HTTP requests to other services when doing service level tests.
As such, I updated the `passthrough` parameter as seen in the code below:

```python
with aioresponses(passthrough=(service_host)) as responses:
    yield responses
```

As you can probably notice, I forgot to add the final comma...

So, now instead of providing a tuple to `passthrough` we were providing a string. However, since a
string can also be iterated the code wasn't broken per say.  When running the test suite no errors
were thrown but the test suite didn't run successfully.

When looking at `aioresponse`'s codebase we noticed that it iterates through the `passthrough` value
and checks if the current URL starts with any of the values in `passthrough`.

**Since we only passed a string, when iterating through `passthrough` the first value will be
`"h"`**. Can you see where this is going?

I'll leave a snippet here to give you and idea of what I'm talking about:

```python
passthrough = ("http://do_not_mock.com")
current_url = "http://mock_me.com"

for url in passthrough:
  if current_url.startswith(url):
    # Do not mock current_url!
```

Basically no HTTP requests were being mocked because all the URLs used in the test suite start with
`"http"` (_well, that's a first!_) and `"http...".startswith("h")` does return `True`. All of this,
and three hours lost, because of a sneaky comma!!!


## Conclusion

In conclusion, I know there's probably not much of an use for a one element tuple but it's a bit
frustating that it relies on that single command at the end, which a lot of people, like me,
might accidentaly delete while messing around a codebase.

I also know that Python is not the only language to have this kind of "issues" (not really an issue
  but you get what I'm saying) so I'm not trying to bash on Python or stating that
  [aioresponse](https://github.com/pnuckowski/aioresponses) should be updated to handle this scenario,

I just think we could have better tuple syntax in Python so as to avoid this kind of confusion.
