---
title: "Comparing C++ and Rust thread APIs"
date: 2025-11-09
---

C++ and Rust don't have the same threading utilities. I'm going to compare them out of curiosity.

# C++

C++20 added `std::jthread` which automatically requests the thread to stop and joins it before the destructor returns.
I really like this behavior because threads are part of RAII.
The thread is running as long as you keep the jthread object alive.
If you have a tree of structs that contain jthreads, dropping the root object will shut down all the threads spawned by the tree.

For this mechanism to work, C++ had to introduce a cancellation API to notify the thread that it should clean up ASAP.
This API is `std::stop_source` for requesting stop and `std::stop_token` for checking for stop.
A jthread creates and stores a stop source for itself. The corresponding stop token is given to the callback run by the thread as parameter.
The user can also choose to pass a callback with 0 parameters if he doesn't want to handle cancellation.

A surprising omission on the part of the committee is that you can't pass you own constructed stop source.
It would have been useful to be able to cancel multiple threads at once from a single stop source.
At my job, I want to register threads to a shutdown service that will ask all of them to finish before turning off the microcontroller.
If a thread finishes on its own, it will unregister itself from the shutdown service.
But acquiring a stop source and registering the thread is only possible after starting it.
This causes a race condition because the thread could end before it gets registered, so it can't unregister itself anymore.
This can be fixed by wrapping jthread in a custom class that accepts a stop source from the outside, but it defeats the purpose of using jthread in the first place.

The second surprising omission in the threading API is the lack of message passing channels.
I don't know the reason they decided to not implement such an essential tool when other modern programming languages have it.
It's not difficult to implement channels yourself with a queue, a mutex, and a condition variable, but I shouldn't have to.

And the third one is not being able to return a value from a thread.
If you need to reply with a value, you need to create a `std::promise`, pass it to the thread, and wait for the future to complete on the parent thread.
Using your own channel also works if you want to yield multiple values from the thread during its execution.
C++ decided to leave the message passing problem to the user, into or out of the thread.
My guess is that an official implementation would have to synergise with the existing cancellation API, which is hard to implement.


# Rust

Rust can spawn threads by calling `std::thread::spawn()` which returns a `JoinHandle`.
Unlike C++, threads don't auto-join when the handle goes out of scope.
The committee decided to not implement a cancellation API, so they probably thought it would be a bad idea to auto-join when dropped.
Threads have no idea they should finish, so it would result in unwanted deadlocks.

Rust has channels compared to C++, so message passing is a solved issue.
You can also return from a thread.
Calling join is the only way to wait for a thread, so the method returns whatever the thread returns.

One interesting behavior of channels in Rust is that the channel will close if either the sender or receiver object is dropped.
This is the reason waiting for an item is falliable.
It doesn't make sense to wait for an item on a closed channel, so the thread is notified of the error.
This mechanic can be used to implement canncelation.
You share a channel with a thread, and when you want to cancel it you drop the sender.
The thread can wait on the channel for either an item or a channel closed error.
Either one of them will unblock it.
A disadvantage of implementing cancellation in this way is that you need to be able to manually drop the sender at will.
This probably means wrapping the sender in an `Option` so you can replace the inner value with `None`.
And if you want to close it from any thread, you most likely need shared ownership and thread safety.
One solution is `Arc<Mutex<Option<mpsc::Sender<T>>>>`.
And if you don't want to leak this implementation detail just to send a cancel notification, you'll need some more abstraction.

Another solution would be to send a special stop value to the channel instead of relying on the closing behavior.
But then the thread would need to handle both stop and channel closed.
And you still have to solve the the implementation leaking.
No one should care about the channel type if all you want is to send a stop signal.
C++'s stop signal is perfect because it's copiable, thread safe, and its only purpose is to request to stop.

# Conclusion

C++ implements cancellation while omitting message passing.
Rust does the opposite.

I don't think it's impossible to have channels and cancellation synergize with each other, so I made [this repo](https://github.com/roscale/cancellation_rs) to try to implement them from scratch and achieve this goal.
It's written in Rust, but it would be exactly the same code in C++.
