---
date: 2023-11-03
authors: [allann]
categories:
  - .net 8
tags: [testing, TimeProvider]
---

# Avoiding flaky tests with TimeProvider and ITimer

This is an extract from [Andrew Lock's blog][blog].  Jump over and check out his work, he is a very talented individual and has some awesome posts.

## What's the problem with the existing APIs?

Even if you're new to C#, you have likely needed to get "the current time" at some point. The default way to do that is to use the `DateTime.UtcNow` or `DateTimeOffset.UtcNow` static properties. You can call these properties from anywhere in your code so they're an easy thing to reach for wherever you are.

<!-- more -->

???+ info
    In this post I'm not going to get into the differences between `DateTime`/`DateTimeOffset`, `Now`/`UtcNow`, or whether you should actually be using something like [NodaTime][noda]. If you aren't aware of that project you should definitely look at it!

Grabbing the current time this way can be fine, right up until you need to test something. Lets take a simple concrete example. Imagine you're modelling a "support ticket" system, which has the following rules to apply when a new ticket is created.

- Record the UTC time the ticket was opened
- If opened by a standard level customer, automatically close the ticket after 7 days.
- It opened by a gold level customer, automatically close the ticket after 14 days.

```cs
public class Ticket
{
    private readonly DateTime _createdDate;
    private readonly DateTime _expiresDate;
    public Ticket(Level level)
    {
        _createdDate = DateTime.UtcNow;
        var expiresAfter = level == Level.Gold ? 14 : 7;
        _expiresDate = _createdDate.AddDays(expiresAfter);
    }

    public bool IsExpired() => _expiresDate > DateTime.UtcNow;
}

public enum Level
{
    Standard,
    Gold,
}
```

This is a naively written type, but lets say you wanted to test it. Currently the only way to test this code is to wait 14 days to make sure the Ticket has expired! There's a very simple solution to this: pass the current date in as a parameter of the methods instead:

```cs
public class Ticket
{
    private readonly DateTime _createdDate;
    private readonly DateTime _expiresDate;
    public Ticket(Level level, DateTime createdDate)
    {
        _createdDate = createdDate;
        var expiresAfter = level == Level.Gold ? 14 : 7;
        _expiresDate = _createdDate.AddDays(expiresAfter);
    }

    public bool IsExpired(DateTime now) => _expiresDate > now;
}
```

This code is now easily testable; you can make sure each **Level** of customer has the expected expiry date by passing in different values for `now` into `IsExpired()`. But what if this test is part of an integration test, so you can't directly manipulate those method arguments?

Another distinct use case is when you want to test something that runs in the background on some sort of timer. For example, consider the following toy example that increments an integer every second:

```cs
public class MyBackgroundWorker : IAsyncDisposable
{
    private readonly TaskCompletionSource<bool> _processExit = new();
    private readonly Task _loopTask;
    
    public int Value { get; set; }
    public DateTimeOffset LastUpdate {get;set;}

    public MyBackgroundWorker()
    {
        _loopTask = Task.Run(RunLoopAsync);
    }

    public async ValueTask DisposeAsync()
    {
        _processExit.TrySetResult(true);
        await _loopTask;
    }

    async Task RunLoopAsync()
    {
        while (!_processExit.Task.IsCompleted)
        {
            // Wait for the delay to expire or disposal
            var delay = Task.Delay(TimeSpan.FromSeconds(1));
            await Task.WhenAny(delay, _processExit.Task);

            Value++;   
            LastUpdate = DateTimeOffset.UtcNow;
        }
    }
}
```

The details of this aren't very important; the key is that we're using a `Task.Delay` to increment the integer periodically. We could also have used a `Timer`, but the overall design is much the same.

Lets say we want to test this. We have a couple of problems:

- _The timespan is fixed_. That will make for slow tests. We could control this by injecting in the delay to use in the constructor, but maybe we don't really want it to be configurable, and it's important that it's _always_ 1 second
- _Using timers in tests often causes flaky tests_. Even if we allow injecting the delay in the constructor, the runtime doesn't guarantee that exactly the requested time will pass for a `Task.Delay()` call, just that at least that time will pass. This is often noticeable in CI where machines are working at maximum capacity; timers don't fire as regularly as you expect, and so tests fail and flake. You can work around this by, for example, adding callbacks inside the loop, but now you're making a lot of changes and adding a lot of API surface you don't necessarily want to expose.

For example, to check that after 5 iterations we have the correct value, our current best effort might look something like this:

```cs
public class WorkerTests
{
    [Fact]
    public async Task BackgroundWorkerRunsPeriodically()
    {
        var delay = TimeSpan.FromSeconds(5);
        var expectedLastUpdate = DateTimeOffset.UtcNow.Add(delay);
        
        var worker = new MyBackgroundWorker();

        // Nominally wait 5s (i.e. 5x1s loop)
        await Task.Delay(delay); 
        await worker.DisposeAsync();

        // Realistically, this might be 4, 5, 6, or 😅
        worker.Value.Should().Be(5);
        // Who knows, this will probably be very flaky 
        worker.LastUpdate.Should().BeCloseTo(expectedLastUpdate, TimeSpan.FromMilliseconds(100));
    }
}
```

Tests like this will always be slow (it takes at least 5s to run) and horribly flaky.

Neither of the design problems I described are insurmountable. We can remove the flake by taking the concerns into account when we design our classes, but they require us to change our implementations, potentially significantly. The new `TimeProvider` abstraction added to .NET 8 provides an alternative solution.

## The TimeProvider abstraction in .NET 8

Adding a "time" abstraction that wraps calls to `DateTime.UtcNow` and other similar APIs has been a common approach to testing for a long time. I've seen such abstractions (often called `IClock`, `ISystemClock`, or `IDateTimeProvider` etc) that look something like this:

```cs
public interface IClock
{
    DateTime UtcNow { get; }
}
```

These abstractions can have simple "default" implementations in your app code:

```cs
public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

but in your tests, you can create fakes that carefully control the values returned as required by specific tests.

.NET 8 introduced a similar abstraction, an abstract base class, called `TimeProvider`:

```cs
public abstract class TimeProvider
{
    public static TimeProvider System { get; }

    protected TimeProvider();

    public virtual TimeZoneInfo LocalTimeZone { get; }
    public virtual long TimestampFrequency { get; }

    public DateTimeOffset GetLocalNow();
    public virtual DateTimeOffset GetUtcNow();
    public virtual long GetTimestamp();
    public TimeSpan GetElapsedTime(long startingTimestamp);
    public TimeSpan GetElapsedTime(long startingTimestamp, long endingTimestamp);

    public virtual ITimer CreateTimer(TimerCallback callback, object? state,TimeSpan dueTime, TimeSpan period);
}
```

This abstraction is similar to the basic `IClock` I showed earlier in that it has the `GetUtcNow()` method, and the `static System` property exposes the default `SystemTimeProvider` implementation that you'll inject into your app in normal operation.

But this abstraction is much bigger than `IClock`, or any other time abstraction I have used before! `TimeProvider` seems to include everything and the kitchen sink—it not only has methods for getting the local or UTC time now, it has methods for comparing timestamps, and even for creating timers! On first thought, that seems like overkill, like a mixing of concerns, but (unsurprisingly) there's a very good reason for it.

The simple `IClock` abstraction allows you to remove the implicit dependency on the system time (i.e. `DateTime.UtcNow` calls) from your own code, but it doesn't do anything about the implicit dependency on the passage of time in the .NET base class library calls. Things like `Task.Delay()` have an implicit dependency on the system time, because they essentially track the difference between two times.

### The ITimer abstraction in .NET 8

One of the important methods on the `TimeProvider` abstraction in .NET 8 is `CreateTimer()`. This method means you can now fully test your `Timer` or `Task.Delay`-based code by refactoring it to use `TimeProvider` instead. The `ITimer` created by `CreateTimer()` is tied to the `TimeProvider` so that the timers trigger as time advances.

What's more, a bunch of overloads have been added to types like Task to allow passing in a `TimeProvider` instance, for example:

```cs
public partial class Task : IAsyncResult, IDisposable
{
    public static Task Delay(TimeSpan delay, TimeProvider timeProvider)
}

public partial class Task<TResult> : Task
{
    public new Task<TResult> WaitAsync(TimeSpan timeout, TimeProvider timeProvider)
}
```

What's more, a new NuGet package, [Microsoft.Bcl.TimeProvider][bcl] backports these APIs all the way back to `netstandard2.0` (and **.NET Framework**) by adding extension method equivalents (as well as the `TimeProvider` and `ITimer` implementations themselves):

```cs
public static class TimeProviderTaskExtensions
{
    public static Task Delay(this TimeProvider timeProvider, TimeSpan delay) 
    public static Task<TResult> WaitAsync<TResult>(this Task<TResult> task, TimeSpan timeout, TimeProvider timeProvider)
}
```

So with all that in mind, lets re-write the `MyBackgroundWorker` to use the `TimeProvider` abstraction and the timer-related overloads.

```cs
public class MyBackgroundWorker : IAsyncDisposable
{
    // Take a dependency on TimeProvider 👇
    private readonly TimeProvider _timeProvider;
    private readonly TaskCompletionSource<bool> _processExit = new();
    private readonly Task _loopTask;
    
    public int Value { get; set; }
    public DateTimeOffset LastUpdate {get;set;}

    // Inject the TimeProvider 👇
    public MyBackgroundWorker(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
        _loopTask = Task.Run(RunLoopAsync);
    }

    public async ValueTask DisposeAsync()
    {
        _processExit.TrySetResult(true);
        await _loopTask;
    }

    async Task RunLoopAsync()
    {
        while (!_processExit.Task.IsCompleted)
        {
                            // Pass TimeProvider into Task.Delay 👇
            var delay = Task.Delay(TimeSpan.FromSeconds(1), _timeProvider);
            await Task.WhenAny(delay, _processExit.Task);

            Value++;   
            // 👇 Use TimeProvider to get the current time 
            LastUpdate = _timeProvider.GetUtcNow();
        }
    }
}
```

If we pass the system TimeProvider into our class, then it behaves exactly as it did before. But now, crucially, we can easily (quickly and reliably) test our class, without having to change its public API.

### Testing with the Microsoft.Extensions.TimeProvider.Testing library

As well as the [Microsoft.Bcl.TimeProvider][bcl] NuGet package, Microsoft have also created the [Microsoft.Extensions.TimeProvider.Testing][test] package for testing. This includes a `TimeProvider` implementation called `FakeTimeProvider`.

`FakeTimeProvider` is a time provider in which you control the flow of time. You can specify a specific start time for the `FakeTimeProvider` and it will act as if that's the current time. You can then call `Advance(TimeSpan)` to advance time forward. After advancing, any `ITimer`s created by the provider will trigger, and the new time will be returned.

All this means that it's relatively easy to create tests thatW test types that rely on timers and accessing the system time, as long as these types use the `TimeProvider` abstraction. For example, we could re-write our background worker test to the following:

```cs
public class WorkerTests
{
    [Fact]
    public async Task BackgroundWorkerRunsPeriodically()
    {
        // We don't care about the start time in this case, but
        // you can set it if you need to
        var fakeTime = new FakeTimeProvider(startDateTime: DateTimeOffset.UtcNow);
        
        // Pass in the FakeTimeProvider as a TimeProvider
        var worker = new MyBackgroundWorker(fakeTime);

        // Advance 0.5s, the timer should not have fired yet (fires every 1s)
        fakeTime.Advance(TimeSpan.FromMilliseconds(500));
        worker.Value.Should().Be(0);
        
        // Forward 1.5s, this will fire the timer twice, but our loop code will only execute once
        fakeTime.Advance(TimeSpan.FromMilliseconds(1_500));
        worker.Value.Should().Be(1); // looped once
        worker.LastUpdate.Should().Be(fakeTime.Start.AddSeconds(2)); // total expired time

        // Forward 1s, we're at 3s total, so the timer should have have fired again
        fakeTime.Advance(TimeSpan.FromSeconds(1));
        worker.Value.Should().Be(2); // looped again
        worker.LastUpdate.Should().Be(fakeTime.Start.AddSeconds(3)); // fired at 3s

        await worker.DisposeAsync();

        // Should have looped one more time on exit
        worker.Value.Should().Be(3);
    }
}
```

This test code shows a couple of features:

- We can advance the clock manually by calling `Advance()`. This controls the value of `GetUtcNow()` and other related methods.
- Advancing the clock triggered the `ITimer`s to fire if they have expired. Note that for this example the `FakeTimeProvider` does not "interrupt" the `Advance` to fire the timer used by `Task.Delay` after 1s; it fires the timer twice at the end of the `Advance()` call. However, the `Task.Delay()` call only "cares" about the first firing, so the second one is "ignored". So we only loop once, and only increment `Value` once, even though we theoretically would expect to fire twice for the time we advanced.

!!! note
    Another cool feature of the `FakeTimeProvider` is `AutoAdvance`. This lets you automatically advance the current time every time someone calls `GetUtcNow()`.

Even with the control `FakeTimeProvider` gives, it's still easy to make mistakes with timers.Even ignoring the oddity around large time advancements, there's actually still a race condition in our test above: our test code might call the first `Advance()` in the test code before the `Task.Delay()` call in the worker, in which case the test above will fail. One possible solution is to simplify the worker implementation to make direct use of the `ITimer` abstraction:

```cs
public class MyBackgroundWorker : IAsyncDisposable
{
    private readonly TimeProvider _timeProvider;
    private readonly ITimer _timer;
    
    public int Value { get; set; }
    public DateTimeOffset LastUpdate {get;set;}

    public MyBackgroundWorker(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;

        // Create a timer directly from the provider
        _timer = timeProvider.CreateTimer(
            callback: UpdateTotals,
            state: this, 
            dueTime: TimeSpan.FromSeconds(1), 
            period: TimeSpan.FromSeconds(1));
    }

    private static void UpdateTotals(object? state)
    {
        // Update the values
        var worker = (MyBackgroundWorker) state!;
        worker.Value++;
        worker.LastUpdate = worker._timeProvider.GetUtcNow();
    }

    public ValueTask DisposeAsync() => _timer.DisposeAsync();
}
```

This implementation is objectively simpler, and as it uses the `CreateTimer()` method directly, it doesn't suffer from the same race condition as the previous implementation. What's more, as we're providing a callback to the `ITimer` directly (instead of indirectly using `Task.Delay()`), our test now does behave correctly when we advance multiple intervals. So if we advance by 3s, our callback will fire 3 times (instead of just once, as in the previous implementation). Lets update the test to account for that:

```cs
public class WorkerTests
{
    [Fact]
    public async Task BackgroundWorkerRunsPeriodically()
    {
        var fakeTime = new FakeTimeProvider(startDateTime: DateTimeOffset.UtcNow);
        var worker = new MyBackgroundWorker(fakeTime);
        await Task.Yield();

        // Advance 0.5s, the timer should not have fired yet
        fakeTime.Advance(TimeSpan.FromMilliseconds(500));
        worker.Value.Should().Be(0);
        
        // Advance another 0.5s and the timer fires!
        fakeTime.Advance(TimeSpan.FromMilliseconds(500));
        worker.Value.Should().Be(1);
        worker.LastUpdate.Should().Be(fakeTime.Start.AddSeconds(1)); // total expired time
        
        // Forward 2s, this fires twice, because we used the ITimer directly
        fakeTime.Advance(TimeSpan.FromSeconds(2));
        worker.Value.Should().Be(3); // fired twice
        worker.LastUpdate.Should().Be(fakeTime.Start.AddSeconds(3)); // total expired time

        // Forward 1s, we're at 4s total, so the timer should have have fired again
        fakeTime.Advance(TimeSpan.FromSeconds(1));
        worker.Value.Should().Be(4); // fired again
        worker.LastUpdate.Should().Be(fakeTime.Start.AddSeconds(4)); // final time

        await worker.DisposeAsync();
    }
}
```

And there we have it! A deterministic test for a class that uses timers. We didn't need to change the public API to test it (outside of using TimeProvider), and yet we can run detailed time-based tests, controlling the flow of time. I feel like this is a game changer for testability, TimeProvider is my new best friend when writing this sort of code!

## Summary

In this post I described some of the reasons a time abstraction can be both useful and/or necessary if you want to test your applications. .NET 8 adds a new abstraction (and implementation) called `TimeProvider` that serves this purpose. As well as providing an abstraction equivalent of `DateTimeOffset.UtcNow`, it also allows creating `ITimer` timer implementations. Moreover, many base library types like `Task` have been updated to work with `TimeProvider` and `ITimer`. These abstractions have even been backported to `netstandard2.0` and **.NET Framework**.

In addition to the abstraction, a testing package, [Microsoft.Extensions.TimeProvider.Testing][test], has been created. This provides a `FakeTimeProvider` which can be used to control the flow of time. This makes it easy to test all your types that depend on the system time or timers internally!

[noda]: https://nodatime.org
[bcl]: https://www.nuget.org/packages/Microsoft.Bcl.TimeProvider
[test]: https://www.nuget.org/packages/Microsoft.Extensions.TimeProvider.Testing
[blog]: https://andrewlock.net/exploring-the-dotnet-8-preview-avoiding-flaky-tests-with-timeprovider-and-itimer