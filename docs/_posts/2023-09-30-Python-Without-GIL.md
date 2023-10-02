# The NoGIL Python proposal , its performance improvement and release timeline


Over the past two decades, Python has consistently held its position as one of the leading programming languages. It has always been my go-to choice for side projects. Python's user-friendly data types such as lists, dictionaries, sets, and strings, coupled with its intuitive syntax resembling natural language, make it both easy to grasp and straightforward to write control logic with. This attribute has played a pivotal role in Python's swift adoption within the data science community, especially in light of the rapid growth in data science and subsequent advancements in machine learning technology.


## What is GIL 
For those who use Python, you may eventually come across the term GIL, standing for Global Interpreter Lock. Understanding the GIL is a common milestone for Python users. However, it's important to note that it doesn't significantly impact your everyday use of Python as a scripting language. If you do encounter issues related to the GIL, it's likely that you're working on either CPU-intensive tasks or I/O-intensive tasks.

In essence, the GIL serves as a Mutex (or Lock) that shields the Python interpreter. This means that at any given moment, only one thread can access the Python interpreter. This poses a considerable challenge when attempting to harness the power of multiple CPU cores in your hardware setup, as only one thread can be in execution at a time, utilizing just one CPU core. This limitation becomes particularly evident in machine learning applications where tasks involve either computationally intensive operations like matrix calculations or I/O-intensive processes such as loading training data and managing intermediate results. The machine learning community has grappled with this constraint.

## Attempts to kill GIL

Throughout history, there have been various endeavors to eliminate the Global Interpreter Lock (GIL) in the Python interpreter. 

In 1999, Greg Stein spearheaded the [free-threading initiative][1]. While it achieved success in terms of functionality, it unfortunately led to a significant slowdown in Python's performance, rendering it impractical for widespread adoption within the community.

 In 2015, Larry Hastings embarked on [the Gelectomy project](https://pythoncapi.readthedocs.io/gilectomy.html), which, once again, encountered performance issues.

Fast forward to PyCon in 2022, where Sam Gross introduced [his "nogil" proposal](https://docs.google.com/document/d/18CXhDb1ygxg-YXNBJNzfzZsDFosB5e6BfnXLlejd9l0/edit#heading=h.kcngwrty1lv), subsequently formalized as ["PEP 703" under the title "Making the Global Interpreter Lock Optional in CPython."](https://peps.python.org/pep-0703/) Following extensive and thorough discussions, on July 28, 2023, the Python steering council made the momentous decision to integrate the nogil work into the official CPython release. This marked a pivotal milestone as Python, after nearly three decades since its inception, took a significant step toward addressing its most notorious characteristic.


Sam Gross is a Software engineer at Meta, he is also working on the PyTorch project. 

## The design decisions of PEP 703
Python relies on reference counting for its Garbage Collection mechanism. This implementation is heavily intertwined with the Global Interpreter Lock (GIL), which poses a significant challenge in attempting to remove it.

In PEP 703, a series of methods have been employed to enhance Python's Garbage Collection to function properly after the GIL is removed:

1. Introduction of biased reference counting: This approach replaces traditional reference counting. Biased reference counting assumes that the majority of variables are confined to a single thread. Each variable is assigned to a specific thread as its owner. It maintains a local variable to track references within this thread and utilizes an atomic integer to monitor references in other threads. These counts are later consolidated when the owning thread's reference count reaches zero.
2. Overhauling the implementation of Python collections (e.g., lists, dictionaries, etc.): PEP 703 dedicates substantial efforts to upgrade these data structures to ensure optimal performance for read-only access in multi-threaded environments.
3. Adoption of a new memory allocator, [mimalloc](https://www.microsoft.com/en-us/research/uploads/prod/2019/06/mimalloc-tr-v1.pdf): This allocator supersedes Python's default pymalloc. Mimalloc is thread-safe and exhibits superior performance, whereas pymalloc lacks thread safety.
   
There are other efforts in PEP 703, but these are the three most important design decisions. 

## The timeline to launch non-GIL python
So here is the question, how soon could we use the non-gil python in our day to day job.  The short answer is that it takes nearly 5-7 years.  The adoption of this effort takes [three steps](https://peps.python.org/pep-0703/#python-build-modes).

> 1. In 2024, CPython 3.13 is released with support for a --disable-gil build time flag. There are two ABIs for CPython, one with the GIL and one without. Extension authors target both ABIs.
> 2. After 2–3 releases, (i.e., in 2026–2027), CPython is released with the GIL controlled by a runtime environment variable or flag. The GIL is enabled by default. There is only a single ABI.
> 3. After another 2–3 release (i.e., 2028–2030), CPython switches to the GIL being disabled by default. The GIL can still be enabled at runtime via an environment variable or command line flag.

![non-gil timeline](/assets/non-gil-timeline.png)

It's anticipated to take approximately 7 years and span 7 releases before non-GIL Python becomes the default option on your MacBook. This extended timeline arises from the fact that a programming language forms the bedrock of the software ecosystem. While adding new features is relatively straightforward, implementing changes that break backward compatibility demands substantial effort. You must wait for all libraries to embrace the modification. A notable example is the transition from Python 2 to Python 3. Python 3.0, also known as "Python 3000" or "Py3K," was launched on December 3, 2008. Despite 15 years passing since its release, there are still a significant number of Python 2 users.

## The performance improvement of non-GIL python

In this section, we're going to conduct benchmarks to evaluate the performance of non-GIL Python.

At present, non-GIL Python can be accessed through two GitHub repositories, which are detailed below:

- https://github.com/colesbury/nogil-3.12
- https://github.com/colesbury/nogil

To install it, simply use the following command:

    pyenv install nogil-3.9.10-1

### benchmark 1, Fibonacci function

The 'nogil' repository offers an optimal code snippet for benchmarking multi-threaded performance. It involves a recursive Fibonacci function without cache memorization. The code snippet is provided below:


    import sys
    from concurrent.futures import ThreadPoolExecutor

    print(f"nogil={getattr(sys.flags, 'nogil', False)}")

    def fib(n):
        if n < 2: return 1
        return fib(n-1) + fib(n-2)

    threads = 8
    if len(sys.argv) > 1:
        threads = int(sys.argv[1])

    with ThreadPoolExecutor(max_workers=threads) as executor:
        for _ in range(threads):
            executor.submit(lambda: print(fib(34)))


I run the tests on my MacBook, equipped with 12 CPU cores and large enough memory. I evaluated the code in both non-GIL Python and GIL Python environments. The outcomes are outlined below:

![benchmark-1](/assets/benchmark-1.png)

As the metrics show,  non-GIL python is nearly 15 times faster than GIL python if we have 60 threads running their own workloads independently, this examples shows the huge negative impact of GIL on python in a multi-thread environment. 

### benchmark 2, Huge List
I also made some adjustments to the test code above. I substituted the Fibonacci function with a more resource-intensive one that involves costly list appends and memory accesses. The modified code is provided below: 

    import sys
    from concurrent.futures import ThreadPoolExecutor

    print(f"nogil={getattr(sys.flags, 'nogil', False)}")

    def list_benchmark(limit):
        x = 0
        for i in range(limit):
            x+=1000
            y = [i for i in range(x)]

    threads = 8
    if len(sys.argv) > 1:
        threads = int(sys.argv[1])

    with ThreadPoolExecutor(max_workers=threads) as executor:
        for _ in range(threads):
            executor.submit(lambda: print(list_benchmark(300)))

Then I ran tests in both GIL python and non-GIL python,  the result is in the table below.  As the chart shows,  the non-GIL python is 7 times faster than the GIL python. 

![benchmark-1](/assets/benchmark-2.png)

The benchmarks clearly demonstrate that the non-GIL version of Python effectively harnesses multiple CPU cores through multi-threading, exhibiting a speed boost of approximately 5-8 times compared to the standard Python version.

Furthermore, evaluations on pyperformance benchmarking indicate that the non-GIL implementation surpasses the with-GIL Python by an average 10% of performance improvement.  [source](https://docs.google.com/document/d/18CXhDb1ygxg-YXNBJNzfzZsDFosB5e6BfnXLlejd9l0/edit) 

## Conclusion

Creating this non-GIL version of Python was a substantial undertaking, involving meticulous design and thorough benchmarking. Our tests above affirm that the non-GIL Python genuinely maximizes the potential of multiple CPU cores, particularly in compute-intensive scenarios. However, it's important to note that even though progress is being made, it may take up to 7 years before non-GIL becomes the default in Python. If you want to try non-GIL python before it is fully mature, you need to take some risk and possibly make adjustments to libraries that aren't inherently thread-safe in their Python code. [hobbit-hole][1]


[1]: <https://mail.python.org/pipermail/python-dev/2000-April/003605.html> "free-threading initiative"
[2]: <https://pythoncapi.readthedocs.io/gilectomy.html> "the Gelectomy project"



