+++
Categories = ["Functional Programming"]
Description = "An Intuitive Use-Case For Monadic Bind And Kleisli Composition"
Tags = ["Haskell", "Monads", "Cxx", "Intel TBB"]
date = "2015-05-01T19:49:27+02:00"
title = "An Intuitive Use-Case For Monadic Bind And Kleisli Composition"

+++

In which I accidentally stumble upon Monadic Bind and Kleisli Composition while parallelizing my C++14 code with Intel TBB's pipeline. This is meant to help you connecting concepts from various fields rather than diving into too much theory.


## Motivation

In my [post about a Distributed Search Engine](https://daniel-j-h.github.io/post/distributed-search-nanomsg-bond/) I'm using the so called survey communication pattern, which works roughly as follows:

* Send request to multiple Respondents
* Gather and merge responses

In pseudocode with the function's declarations this may look like:

```
Blob getMemoryBlobFromNetwork();
Response deserializeBlob(Blob);


ResponseSet responses;

sendSurveyRequest("Tell me the current time in CEST");

while not surveyDeadlineReached() {
  Blob blob = getMemoryBlobFromNetwork();
  Response response = deserializeBlob(blob);
  responses.merge(response);
}
```

This works rather well, but there is room for improvement.
Although the Respondents send their responses concurrently, we handle responses one after the other.
That is we first receive a response as memory blob from the networking layer.
We then deserialize it, after which we merge it into a set of responses.
It is not until then that we turn our attention to the next response.

Notice how the next function's input depends on the current function's output.
With this kind of dependency the Pipeline Pattern can be easily introduced, with the goal of running the pipeline's stages in parallel, increasing the system's throughput.

With three responses the pipeline may look as follows:

```bash
      step 1    |     step 2    |     step 3    |     step 4    |     step 5    |
    -----------------------------------------------------------------------------

    receive()   |   receive()   |   receive()   |
                | deserialize() | deserialize() | deserialize() |
                                |    merge()    |    merge()    |    merge()    |
```

You can see how we are now able to receive(), deserialize() and merge() potentially in parallel running on multiple threads.

In the Distributed Search Engine C++14 code this can be accomplished rather easily, using Intel TBB's pipeline. I pushed a [commit](https://github.com/daniel-j-h/DistributedSearch/commit/97224b179fdc050dc219287616e8d3073e0e0a8c) with this to the original Distributed Search Engine repository. See [the code](https://github.com/daniel-j-h/DistributedSearch/blob/97224b179fdc050dc219287616e8d3073e0e0a8c/Service.cc#L110-L114) for yourself.

## Failures And How To Propagate Them

Failures happen. We may not be able to deserialize a request.
Maybe we have additional uncompress or decrypt stages that may fail.
But what happens if an exception occurs in a particular pipeline stage? With the above implementation this would bring down the whole pipeline.

What we really want from our functions is to express a computation that may fail.
That is, instead of the exception throwing functions we want them to wrap their response in something that is able to express failure, such as the generic Optional, also known as Maybe Monad. Here are the declarations with this in mind:

```
  Optional<Blob> getMemoryBlobFromNetwork();
  Optional<Blob> uncompressBlob(Blob blob);
  Optional<Blob> decryptBlob(Blob blob);
  Optional<Response> deserializeBlob(Blob blob);
```

Perfect! But wait, now the types do not match!
Our functions do not expect arguments wrapped in an Optional but instead types such as a raw Blob. What we have to do is convert from Optionals to their value type by checking for success and then passing the value type on:

```
while not surveyDeadlineReached() {
  Optional<Blob> blob0 = getMemoryBlobFromNetwork();
  if (blob0) {
    Optional<Blob> blob1 = decryptBlob(*blob0);

    if (blob1) {
      Optional<Blob> blob2 = uncompressBlob(*blob1);

      if (blob2) {
        Optional<Response> response = deserializeBlob(*blob2);

        if (response) {
          responses.merge(*response);
        }
      }
    }
  }
}
```

This is horrible error handling. There has to be a better way. And there is.
Note: you may flatten this kind of error handling with guards, but the overhead remains.

First let us switch to Haskell for pseudocode as I don't want to show you the C++ that is required for this.
Our dummy functions representing the pipeline stages now take integers and may return an optional of integer, Maybe Integer that is:

```haskell
-- :: Integer -> Maybe Integer
f0 = \x -> Just (x + 1)
f1 = \x -> Just (x + 10)
f2 = \x -> Just (x + 100)
```

And a function that always fails, to provoke errors later on:

```haskell
-- :: Integer -> Maybe Integer
g0 = \_ -> Nothing
```

What we still need is a way to convert our functions taking integers to functions taking a Maybe Integer, checking for success, and only then calling the plain function. That is, the type has to be similar to:

```haskell
Maybe Integer -> (Integer -> Maybe Integer) -> Maybe Integer
```

This looks suspiciously similar to what Monadic Bind does. Take a look:

```haskell
(>>=) :: m a -> (a -> m b) -> m b
```

Let's see it in action, using it for passing integers wrapped as Maybe Integer to our functions taking an integer:

```haskell
Just 0 >>= f0
Just 1

Just 0 >>= f0 >>= f1 >>= f2
Just 111

Just 0 >>= f0 >>= g0 >>= f1 >>= f2
Nothing
```

Monadic Bind for the Maybe monad does exactly what we need:

* in case the passed in Maybe Integer is a Just we go on executing the function on its value
* in case the passed in Maybe Integer is Nothing we do not execute the function but instantaneously return Nothing

This is exactly the semantic we want for our motivating pipeline example!


## Composing Propagating Pipeline Stages

Using Maybe's Monadic Bind as above allows us to adapt our functions without the need for local error handling. Great! But how do we compose a pipeline then?
We could directly use Monadic Bind or think about the types involved.
Say we take two functions and want to compose them:

```haskell
(Integer -> Maybe Integer) -> (Integer -> Maybe Integer) -> Integer -> Maybe Integer
```

That is, with two functions and an integer we start off passing the integer to the first function, then doing our Monadic Bind trick from above, potentially passing the result to the second function, returning the result.

That is what the so called Kleisli composition operator does:
```haskell
(>=>) :: (a -> m b) -> (b -> m c) -> a -> m c
```

Look:

```haskell
pipeline = f0 >=> f1 >=> f2
pipeline :: Integer -> Maybe Integer

Just 1 >>= pipeline
Just 112
```

Equipped with Monadic Bind and Kleisli Composition we may go back to our C++14 code base and implement the ideas there. I'm not going to show you how this can be done here, as it requires some more boilerplate code to make the compiler happy -- as usual.


## Further Remarks


You probably want to propagate an explanation of what exactly went wrong in case of failure.
This can be done switching out the Maybe Integer for an Either PipelineError Integer, with PipelineError being a sum type of the pipeline stage's potential errors.

See Scott Wlaschin's slides on monadic bind and what he calls "Railway Oriented Programming":

* [Functional Programming Patterns (NDC London 2014)](http://www.slideshare.net/ScottWlaschin/fp-patterns-ndc-london2014)
* [Railway Oriented Programming](http://www.slideshare.net/ScottWlaschin/railway-oriented-programming)

For more about monads, see Learn You A Haskell [Chapter 12 and 13](http://learnyouahaskell.com/chapters).

Haskell documentation:

* [Monadic Bind, >>=](https://hackage.haskell.org/package/base-4.6.0.1/docs/Control-Monad.html#v:-62--62--61-)
* [Kleisli Composition, >=>](https://hackage.haskell.org/package/base-4.6.0.1/docs/Control-Monad.html#v:-62--61--62-)


## Summary

In using Monadic Bind and Kleisli Composition we not only composed error propagating pipeline stages, but we did this in a way that keeps the code clean and shows its intention to the reader. Being able to see patterns such as functional composition in code that may look totally unrelated and not in functional style at all -- say C++ in the context of parallelization -- helps tremendously.
