# Every variable shared by multiple thread should be wrapped in a Safe object
## Overview
*safe* is a header-only library that helps you get your multi-threaded code safe and understandable. It defines the Safe and Access classes. A Safe object packs a lockable object (e.g. std::mutex) and a value object (whatever you need to protect using the lockable object). Safe objects expose the value object through a simple, clear and safe interface: use the access() member function set to gain protected access to the value, and use the unsafe() function set for unprotected access. Protected access is achieved through the Access class. Think of the Access class as a combination of a lock (e.g. std::lock_guard) and a pointer to the value object. The lock gives you the full power of RAII for managing the lockable object, and the pointer-like functionality only exists for the span of time where the lockable object is locked.

Here is why you want to use safe:
### Without safe
```c++
std::mutex frontEndMutex;
std::mutex backEndMutex;
int value; // <-- do I need to lock a mutex to safely access this value ?
{
	std::lock_guard<std::mutex> lock(frontEndMutex); // <-- is this the right mutex ?
	++value;
}
--value; // <-- unprotected access, is this intended ?
```
### With safe
```c++
std::mutex frontEndMutex;
safe::Safe<int> value; // <-- value and mutex packaged together!
{
	auto&& safeValue = value.access(); // <-- right mutex: guaranteed!
	//  ^^ do not mind the rvalue reference, I will explain its presence later on.

	++*safeValue; // access the value using pointer semantics: * and ->
} // from here, you cannot directly access the value anymore: jolly good, since the mutex is not locked anymore!

--value.unsafe(); // <-- unprotected access: clearly expressed!
```
### Vocabulary
* *safe*: the safe library.
* Value object: whatever needs to be protected by a lockable object.
* Lockable object: an object that exhibits the BasicLockable interface: lock() and unlock(). Examples are std::mutex and std::recursive_mutex.
* Safe object: combines a value object and a lockable object. Exposes the value object through a simple and expressive interface.
* Lock object: an object that manages a lockable object. Examples are std::lock_guard and std::unique_lock.
* Access object: a Lock object that also gives pointer-like access to the value object.
* Access mode: Access objects can be created with read-write or read-only behavior. Read-only Access objects are especially useful to enforce the read-only nature of std::shared_lock (C++14) and boost::shared_lock_guard.
## Main features:
### 1. Safety and clarity
No more locking the wrong mutex, no more mistaken access outside the safety of a locked mutex.  
No more naked shared variables, no more plain mutexes lying around, no more mutable (ever used a member mutex variable within a const-qualified member function ?).
### 2. Flexibility
#### Choose the lockable and lock that fit your need
The Safe class template has a template parameter for the lockable object: use std::mutex, std::shared_mutex (C++17), name it!  
The Access class template has a template parameter for the lock object: use std::lock_guard, boost::shared_lock_guard, anything you want! You can use a different lock for every Access object you construct.
#### Store the value object/lockable object inside the Safe object, or refer to existing objects
You can use any combination of reference and non-reference types for your Safe objects:
```c++
safe::Safe<int, std::mutex>;
safe::Safe<int>; // equivalent to the above, as the second template parameter defaults to std::mutex
safe::Safe<int&, std::mutex>;
safe::Safe<int, std::mutex&>;
safe::Safe<int&, std::mutex&>;
```
#### Flexibly construct the value and lockable objects
Just remember, the first argument to a Safe constructor is used to construct the lockable object, the other arguments are used for the value object.
*Note: when constructing a Safe object and the lockable object is default constructed but the value object is not, you must pass the safe::default_construct_lockable tag.*
Examples:
```c++
std::mutex aMutex;

safe::Safe<int, std::mutex> bothDefault; // lockable and value are default constructed
safe::Safe<int, std::mutex&> noDefault(aMutex, 42); // lockable and value are initialized
safe::Safe<int, std::mutex&> valueDefault(aMutex); // lockable is initialized, and value is default constructed
safe::Safe<int, std::mutex> lockableDefault(safe::default_construct_lockable, 42); // lockable is default constructed, and value is initialized
```
#### Flexibly construct the Lock objects
Both the Access constructors and the Safe::access member functions use a variadic parameter that is forwarded to the Lock constructor. This can be used to pass in standard lock tags such as std::adopt_lock, but also to construct you custom locks that may require additionnal arguments than just the lockable object.
```c++
safe::Safe<int> value;
value.lockable().lock();
{
	safe::Safe<int>::Access<> safeValue(value, std::adopt_lock);
}
{
	auto&& safeValue = value.access(std::adopt_lock);
}
```
### 3. Even more safety!
#### Choose the access mode that suits each access
Once you construct a Safe object, you fix the type of the lockable object you will use. From there, you will create an Access object every time you want to operate on the value object. For each of these accesses, you can choose whether the access is read-write or read-only.
#### Force read-only access with shared mutexes and shared_locks
Shared mutex and shared locks allow multiple reading threads to access the value object simultaneously. Unfortunately, using only mutexes and locks, the read-only restriction is not guaranteed to be applied. That is, it is easy to create a situation where a thread locks a mutex in shared mode and writes to the shared value. With safe, you can enforce read-only access when using shared locking.
### 4. Compatibility
#### With legacy code
You can use *safe* with old-style unsafe code that use the out-of-fashion separate mutex and value idiom:
```c++
// Unsafe code below!
std::mutex mutex;
int value;
// End of usafe code!
// Safe code from here on
safe::Safe<int&, std::mutex&> safeValue(mutex, value);
// Forget about mutex and value, only use safeValue
```
#### With code from the future
*safe* is written in C++11, but it is fully compatible with lockable and lock from different sources like C++14's std::shared_lock and C++17's std::shared_mutex, thanks to template parameters. Of course, you can also use boost::shared_lock_guard and your own custom lockables and locks (see the lockonce class from my *mess* repository).
#### With standard uses of locks and lockables
The lockable object is accessible from the Safe object through accessor functions, and the lock object is a public member of the Access class. Anything you can do with your typical mutexes and locks you can do with *safe*. For example, *safe* can seamlessly be used with std::condition_variable:
```c++
std::condition_variable cv;
safe::Safe<int> value;
auto safeValue = value.access<std::unique_lock>();
//  ^ oh, the rvalue reference dissapeared, keep reading...
cv.wait(safeValue.lock);
```
## Basic usage
### Include the header-only library
```c++
#include "safe.hpp"
```
### Replace your value and lockable objects by a Safe object
```c++
// std::mutex mutex;
// int value;
safe::Safe<int> value;
```
### Replace your lock objects by Access objects
```c++
// std::lock_guard<std::mutex> lock(mutex);
auto&& safeValue = value.access();
//  ^^ argh, this rvalue reference again! Here is the explanation:
```
#### Why the rvalue reference ?
The Access\<std::lock_guard\> object returned by a call to Safe::access() is non-copyable, non-moveable because it aggregates a non-copyable, non-moveable std::lock_guard object. Thus, it cannot be captured with a typical syntax, an rvalue reference must be used:
```c++
safe::Safe<int> value;
// auto safeValue = value.access(); // <-- does not compile!
auto&& safeValueLockGuard = value.access(); // need rvalue reference
auto safeValueUniqueLock = value.access<std::unique_lock>(); // no rvalue reference needed
```
The ravlue reference is totally safe: it extends the lifetime of the object it refers too. Altough slightly annoying, it is the right thing to do pre-C++17. With C++17 and later, it is no longer necessary to use the rvalue reference.
### Access your value object though the Access objects using pointer semantics
```c++
// value = 42;
*safeValue = 42;
```
## Going a little bit deeper
### One-liners
The Safe::access() member functions can be used to write safe and compact one-liners:
```c++
safe::Safe<std::vector<int>> vector;
// One-liner to assign a new value to the vector
*vector.access() = std::vector<int>(1, 2);
// One-liner to clear the vector
vector.access()->clear();
```
### Specifying the access mode
The Access class has one template parameter for the lock type, and another one for the access mode. Likewise, overloads of the Safe::access() member functions exist with a parameter for the lock type and one for the access mode. The AccessMode template parameter is an enum which can take two possible values: ReadOnly and ReadWrite. Here are examples of read-only Access objects:
```c++
safe::Safe<int> safeValue;
safe::Safe<int>::Access<std::lock_guard, safe::eAccessModes::ReadOnly> value(safeValue);
auto&& sameValue = safeValue.access<safe::eAccessModes::ReadOnly>();
```
## Advanced use cases
### Specifying a default access mode for a lock type, and enforcing read-only access
Here is the full declaration of the Access class:
```c++
template<template<typename> class LockType = std::lock_guard, eAccessModes AccessMode = LockTraits<LockType>::IsReadOnly ? eAccessModes::ReadOnly : eAccessModes::ReadWrite> class Access
```
Don't be daunted be the first template parameter, it only means that the LockType must be a class template with one template parameter (like std::lock_guard and std::unique_lock).  
Now don't be daunted by the second template parameter, it is the customization point you can use to change the default bahevior of the Access class. As already mentioned, the AccessMode template parameter defines the access mode for the Access class: either read-write or read-only. The parameter has a default value, and this default value depends on the LockTraits type trait that is defined in safetraits.h. If no specialization of the type trait exists for LockType, the default access mode is ReadWrite. If a specialization exists, it must declare the IsReadOnly boolean variable which determines the default access mode for the lock type. If IsReadOnly is true, the default value for AccessMode is eAccessModes::ReadOnly *and* using AccessMode = eAccessModes::ReadWrite will fail to compile. If IsReadOnly is false, the default value for AccessMode is eAccessModes::ReadWrite.  
It is useful to specify a ReadOnly default access mode for shared locks like std::shared_unique_lock. Example:
```c++
template<>
struct LockTraits<std::shared_lock>
{
	static constexpr bool IsReadOnly = true;
};
```
### Using safe in templated code
Writing templated code using safe reveals a little bit too much the true nature of C++ because Access is a template class the Safe::access() member functions are template functions, and both are defined inside the Safe class template. It means that if you use safe in templated code (i.e. if you do not specify a value for all the template parameters of the Safe class template), you will need to use the following convoluted syntax (**notice the "::template " and ".template " syntax**):
```c++
template <typename ValueType>
class Example
{
public:
	safe::Safe<ValueType> m_safeValue;

	void exampleAccessType()
	{
		typename safe::Safe<ValueType>::template Access<std::lock_guard> value(m_safeValue);
	}
	void exampleAccessMemberFunction()
	{
		auto&& value = m_safeValue.template access<std::lock_guard>();
	}
};
```
### Returning an Access<std::lock_guard> object
Most of the time when you use safe, you will have a Safe object as a private member variable of your class and use it to access its value object in a safe way. Example:
```c++
#include "safe.hpp"

class MultithreadedCount
{
public:
	void increment()
	{
		++*m_count.access();
	}

private:
	safe::Safe<int> m_count;
};
```
When client code calls the increment() function, you lock the Safe object's mutex using an Access\<std::lock_guard\> object, and then increment the value. This is all nice and good, but imagine you would like client code to do more than just increment the count ? Will you write a function for every operation that exist on an int ? Even if you do, the locking and unlocking will be too fine-grained, as it will happen at every call of a function. The right thing to do is to write one function that returns an Access\<std::lock_guard\> object:
```c++
	safe::Safe<int>::Access<> get() // Access<> defaults to std::lock_guard and ReadWrite template parameters
	{
		return {m_count};
	}
```
**Notice the list-initialization in the return statement**. Pre-C++17, it is mandatory to use this syntax, again because Access\<std::lock_guard\> is non-copyable, non-moveable.

With such an interface, the client has a lot of freedom and the guarantee of thread safety.
```c++
MultithreadedCount count;
count.increment(); // thread safety managed inside the function call, simple but limited
{
	// thread safety managed by the access variable, efficient and flexible!
	auto&& safeCount = count.get(); // capture the Access<std::lock_guard> object by rvalue reference
	// do anything you like with the variable!
	*safeCount = 43; 
	--*safeCount;
} // mutex automatically unlocked when scope is exited
```
See the State class in my *mess* repository for a concrete example of this technique.
## Complete examples
The State and Resource classes from my *mess* repository use *safe*.
