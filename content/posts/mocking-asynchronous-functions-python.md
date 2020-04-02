---
title: "Mocking Asynchronous Functions In Python"
date: 2020-02-18T22:12:15Z
draft: false
description: "Post on how to mock asynchronous (asyncio) functions in Python."
tags: ["Python"]
---

## Introduction

You might have already heard about Python's `asyncio` module, it allows you to easily run concurrent
code using Python.
In the last few months I've worked in some codebases which take advantage of the benefits of
`asyncio`, mainly through their use of [aiohttp](https://docs.aiohttp.org/en/stable/).

When using `asyncio` you'll use the `async/await` keywords to both define and call
asynchronous functions.
This also means changes in the way you test your code because, unlike ordinary functions,
asynchronous functions always return a coroutine object, which needs to be awaited, using the
`await` keyword in order to actually schedule it, run it and get the actual return value.

As such, let's take a quick look into how we can easily test asynchronous functions by leveraging
[Futures](https://docs.python.org/3/library/concurrent.futures.html) in Python 3.7 and
[AsyncMock](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.AsyncMock)
in Python 3.8

## What we're mocking

In this example, we're going to be mocking a simple function which adds two integers, its parameters,
while resorting to `asyncio.sleep` to simulate IO heavy tasks, for example, HTTP requests or a
database calls.

```python
import asyncio

async def sum(x, y):
    await asyncio.sleep(1)
    return x + y
```

## Mocking it

Asynchronous functions in Python return what's known as a `Future` object, which contains the
result of calling the asynchronous function.
As such, the "secret" to mocking these functions is to make the patched function return a
`Future` object with the result we're expecting, as one can see in the example below.

```python
import pytest
import asyncio

@pytest.fixture()
def mock_sum(mocker):
    future = asyncio.Future()
    future.set_result(4)
    mocker.patch('app.sum', return_value=future)
```

As you can see in the example above, we're creating a pytest fixture, namely `mock_sum` that
patches the function we created at the beginning of the post and specifies that the function call
will return a `Future` object, with a result of `4`.
In your own tests you will, of course, need to change the call to `set_result` to return whatever
value you're expecting, maybe a HTTP response or some database query result.

With this done we can now create a simple test case that tests the `sum` function:

```python
import pytest import asyncio

@pytest.mark.asyncio
async def test_sum(mock_sum):
    result = await sum(1, 2)
    # I know 1+2 is equal to 3 but one man can only dream!
    assert result == 4
```

There's also a few different things happening here when compared to a regular test function:

- `@pytest.mark.asyncio` decorator - This tells pytest that this is an asynchronous test function,
  otherwise pytest will skip it.
- `async def test_sum(mock_sum)` - Defines the asynchronous test function while at the same time
  calls the pytest fixture, `mock_sum`, so that it successfully mocks the `sum` function's result.
- `result = await sum(1,2)` - Correctly calls the asynchronous function using the `await` keyword.

Although `1 + 2` is equal to `3` I'm purposefully asserting that this returns `4` so as to make sure
that the fixture is indeed called.
If you go ahead and run `pytest` now with the code shown above you should see that indeed it
executes successfully, passing the tests.

However, imagine that you want to mock the `sum` function multiple times while having a different
value provided to `set_result` in the `Future` object. It doesn't make sense to create multiple
fixtures since we'll be repeatedly patching the same function. In this case we'll return the
`Future` object and call the `set_result` function in the test function, thus,
our fixture we'll now look like:

```python
import pytest
import asyncio

@pytest.fixture()
def mock_sum(mocker):
    future = asyncio.Future()
    mocker.patch('app.sum', return_value=future)
    return future
```

Notice that we're not calling `set_result` in the fixture this time around. With the updated
fixture we now need to update the test function to look like this:

```python
import pytest
import app

@pytest.mark.asyncio
async def test_sum(mock_sum):
    mock_sum.set_result(4)
    result = await app.sum(1, 2)
    # I know 1+2 is equal to 3 but one man can only dream!
    assert result == 4
```

Finally, notice now how we're calling `mock_sum.set_result(4)`. If we want the mock to return
different values we now just need to change the value provided to `set_result` instead of having to
create multiple fixture for different tests!

## Mocking It In Python 3.8

The code above only works for versions of Python <3.8. In Python 3.8 we need to change the code
slightly because
[`AsyncMock`](https://docs.python.org/3.8/library/unittest.mock.html#unittest.mock.AsyncMock) has
been introduced.

With that said, we can simply change the mocking function to return the `AsyncMock` instance instead
of the `Future` instance.

```python
from unittest.mock import AsyncMock

@pytest.fixture()
def mock_sum(mocker):
    async_mock = AsyncMock(return_value=4)
    mocker.patch('app.sum', side_effect=async_mock)
```

As you can see in the code above, the main change is that the return value is now set as an
`AsyncMock` instance instead of a `Future` instance, and we can also now use the `return_value`
argument in the `AsyncMock` instantiation instead of needing to call a function afterwards to set
its result.

With the code above our test function will look like the first showed in this blog post, where we
don't change the result of the mock. However, as we did in the end of the previous section, if we
need to mock the same function multiple times while having different results it's better if we
just return the `AsyncMock` instance from the fixture and set the `return_value` in the test
function. As such, our fixture would now look like this:

```python
from unittest.mock import AsyncMock

@pytest.fixture()
def mock_sum(mocker):
    async_mock = AsyncMock()
    mocker.patch('app.sum', side_effect=async_mock)
    return async_mock
```

And with this fixture we could simply update our test function to the following:

```python
@pytest.mark.asyncio
async def test_sum(mock_sum):
    mock_sum.return_value = 4
    result = await app.sum(1, 2)
    assert result == 4
```

Notice that the only change compared to the previous section is that we now set the `return_value`
attribute of the mock instead of calling the `set_result` function seeing as we're now working with
`AsyncMock` instead of `Future`. Aside that, the test function looks exactly the same.

## Conclusion

In conclusion mocking asynchronous functions in Python is actually easier than I expected at first,
mostly because I didn't really understood how asyncio worked. After some reading and experimentation
it turns out it's quick and easy to do, and it allows you to run concurrent tests, which should
speed up your test suite!

If you're reading this for a quick solution and don't really known what's going on I'd advise
[reading up on asyncio](https://realpython.com/async-io-python/).

I hope this blogpost has helped you! 👋

---

* [Reddit Discussion](https://www.reddit.com/r/Python/comments/ffwri2/mocking_asynchronous_functions_in_python_dino_dot/)
* [Hackernews Discussion](https://news.ycombinator.com/item?id=22526405)
