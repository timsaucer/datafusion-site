---
layout: post
title: "Comparing approaches to User Defined Functions in Apache DataFusion using Python"
date: "2024-08-06 00:00:00"
author: timsaucer
categories: [tutorial]
---

<!--
{% comment %}
Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to you under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
{% endcomment %}
-->
# Writing User Defined Functions in Apache DataFusion using Python

## Personal Context

For a few months now I’ve been working with [Apache DataFusion](https://datafusion.apache.org/), a
fast query engine written in rust. From my experience the language that nearly all data scientists
are working in is Python. In general, often stick to [pandas](https://pandas.pydata.org/) for
in-memory tasks and [pyspark](https://spark.apache.org/) for larger tasks that require distributed
processing.

In addition to DataFusion, there is another rust based newcomer to the DataFrame world,
[Polars](https://pola.rs/). It is growing extremely fast, and it serves many of the same use cases
as DataFusion. For my use cases, I'm interested in DataFusion because I want to be able to build
small scale tests rapidly and then scale them up to larger distributed systems with ease. I do
recommend evaluating Polars for in memory work.

Personally, I would love a single query approach that is fast for both in memory usage and can
extend to large batch processing to exploit parallelization. I think DataFusion, coupled with
Ballista, may provide this solution.

As I’m testing, I’m primarily limiting my work to the
[datafusion-python](https://datafusion.apache.org/python/) project, a wrapper around the rust
DataFusion library. This wrapper gives you the speed advantages of keeping all of the data in the
rust implementation and the ergonomics of working in python. Personally, I would prefer to work
purely in rust, but I also recognize that since the industry works in python we should meet the
people where they are.

## User-Defined Functions

The focus of this post is User-Defined Functions (UDFs). The DataFusion library gives a lot of
useful functions already for doing DataFrame manipulation. These are going to be similar to those
you find in other DataFrame libraries. You’ll be able to do simple arithmetic, create substrings of
columns, or find the average value across a group of rows. These cover most of the use cases
you’ll need in a DataFrame.

However, there will always arise times when you want a custom function. With UDFs you open a
world of possibilities of your code. Sometimes there simply isn’t an easy way to use built-in
functions to achieve your goals.

In the following, I’m going to demonstrate two example use cases. These are based on real world
problems I’ve encountered. Also I want to demonstrate the approach of “make it work, make it work
well, make it fast” that is a motto I’ve seen thrown around in data science.

I will demonstrate three approaches to writing UDFs. In order of increasing performance they are

- Writing a pure python function to do your computation
- Using the pyarrow libraries in python to accelerate your processing
- Writing a UDF in rust and exposing it to python

Additionally I will demonstrate two variants of this. The first will be nearly identical to the
pyarrow library approach to simplicity of understanding how to connect the rust code to python. The
second version we will do the iteration through the input arrays ourselves to give even greater
flexibility to the user.

Here are the two example use cases, taken from my own work but generalized.

### Use Case 1: Scalar Function

I have a DataFrame and a list of tuples that I’m interested in. I want to filter out the DataFrame
to only have values that match those tuples from certain columns in the DataFrame. For example,
suppose I have a table of sales line items. There are many columns, but I am interested in three: a
part key, supplier key, and return status. I want only to return a DataFrame with a specific
combination of these three values.

Probably the most ergonomic way to do this without UDF is to turn that list of tuples into a
DataFrame itself, perform a join, and select the columns from the original DataFrame. If we were
working in pyspark we would probably broadcast join the DataFrame created from the tuple list since
it is tiny. In practice, I have found that with some DataFrame libraries performing a filter rather
than a join can be significantly faster. This is worth profiling for your specific use case.

### Use Case 2: Aggregate or Window Function

I have a DataFrame with many values that I want to aggregate. I have already analyzed it and
determined there is a noise level below which I do not want to include in my analysis. I want to
compute a sum of only values that are above my noise threshold.

This can be done fairly easy without leaning on a User Defined Aggegate Function (UDAF). You can
simply filter the DataFrame and then aggregate using the built-in `sum` function. Here, we
demonstrate doing this as a UDF primarily as an example of how to write UDAFs. We will use the
pyarrow compute approach.

## Pure Python approach

The fastest way (developer time, not code time) for me to implement the scalar problem solution
was to do something along the lines of “for each row, check the values of interest contains that
tuple”. I’ve published this as
[an example](https://github.com/apache/datafusion-python/blob/main/examples/python-udf-comparisons.py)
in the [datafusion-python repository](https://github.com/apache/datafusion-python). Here is an
example of how this can be done:

```python
values_of_interest = [
    (1530, 4031, "N"),
    (6530, 1531, "N"),
    (5618, 619, "N"),
    (8118, 8119, "N"),
]

def is_of_interest_impl(
    partkey_arr: pa.Array,
    suppkey_arr: pa.Array,
    returnflag_arr: pa.Array,
) -> pa.Array:
    result = []
    for idx, partkey in enumerate(partkey_arr):
        partkey = partkey.as_py()
        suppkey = suppkey_arr[idx].as_py()
        returnflag = returnflag_arr[idx].as_py()
        value = (partkey, suppkey, returnflag)
        result.append(value in values_of_interest)

    return pa.array(result)

# Wrap our custom function with `datafusion.udf`, annotating expected 
# parameter and return types
is_of_interest = udf(
    is_of_interest_impl,
    [pa.int64(), pa.int64(), pa.utf8()],
    pa.bool_(),
    "stable",
)

df_udf_filter = df_lineitem.filter(
    is_of_interest(col("l_partkey"), col("l_suppkey"), col("l_returnflag"))
)
```

When working with a DataFusion UDF in python, you define your function to take in some number of
expressions. During the evaluation, these will get computed into their corresponding values and
passed to your UDF as a pyarrow Array. We must return an Array also with the same number of
elements (rows). So the UDF example just iterates through all of the arrays and checks to see if
the tuple created from these columns matches any of those that we’re looking for.

I’ll repeat because this is something that tripped me up the first time I wrote a UDF for
datafusion: **DataFusion UDFs, even scalar UDFs, process an array of values at a time not a single
row.** This is different from some other DataFrame libraries and you may need to recognize a slight
change in mentality.

Some important lines here are the lines like `partkey = partkey.as_py()`. When we do this, we pay a
heavy cost. Now instead of keeping the analysis in the rust code, we have to take the values in the
array and convert them over to python objects. In this case we end up getting two numbers and a
string as real python objects, complete with reference counting and all. Also we are iterating
through the array in python rather than rust native. These will **significantly** slow down your
code. Any time you have to cross the barrier where you change values inside the rust arrays into
python objects or vice versa you will pay **heavy** cost in that transformation. You will want to
design your UDFs to avoid this as much as possible.

## Python approach using pyarrow compute

DataFusion uses [Apache Arrow](https://arrow.apache.org/) as its in-memory data format. This can
be seen in the way that Arrow Arrays are passed into the UDFs. We can take advantage of the fact
that [pyarrow](https://arrow.apache.org/docs/python/), the canonical Python Arrow implementation,
provides a variety of
useful functions. In the example below, we are only using a few of the boolean functions and the
equality function. Each of these functions takes two arrays and analyzes them row by row. In the
below example, we shift the logic around a little since we are now operating on an entire array of
values instead of checking a single row ourselves.

```python
import pyarrow.compute as pc

def udf_using_pyarrow_compute_impl(
    partkey_arr: pa.Array,
    suppkey_arr: pa.Array,
    returnflag_arr: pa.Array,
) -> pa.Array:
    results = None
    for partkey, suppkey, returnflag in values_of_interest:
        filtered_partkey_arr = pc.equal(partkey_arr, partkey)
        filtered_suppkey_arr = pc.equal(suppkey_arr, suppkey)
        filtered_returnflag_arr = pc.equal(returnflag_arr, returnflag)

        resultant_arr = pc.and_(filtered_partkey_arr, filtered_suppkey_arr)
        resultant_arr = pc.and_(resultant_arr, filtered_returnflag_arr)

        if results is None:
            results = resultant_arr
        else:
            results = pc.or_(results, resultant_arr)

    return results


udf_using_pyarrow_compute = udf(
    udf_using_pyarrow_compute_impl,
    [pa.int64(), pa.int64(), pa.utf8()],
    pa.bool_(),
    "stable",
)

df_udf_pyarrow_compute = df_lineitem.filter(
    udf_using_pyarrow_compute(col("l_partkey"), col("l_suppkey"), col("l_returnflag"))
)
```

The idea in the code above is that we will iterate through each of the values of interest, which we
expect to be small. For each of the columns, we compare the value of interest to it’s corresponding
array using `pyarrow.compute.equal`. This will give use three boolean arrays. We have a match to
the tuple if we have a row in all three arrays that is true, so we use `pyarrow.compute.and_`. Now
our return value from the UDF needs to include arrays for which any of the values of interest list
of tuples exists, so we take the result from the current loop and perform a `pyarrow.compute.or_`
on it.

From my benchmarking, switching from approach of converting values into python objects to this
approach of using the pyarrow built-in functions leads to about a 10x speed improvement in this
simple problem.

It’s worth noting that almost all of the pyarrow compute functions expect to take one or two arrays
as their arguments. If you need to write a UDF that is evaluating three or more columns, you’ll
need to do something akin to what we’ve shown here.

## Rust UDF with Python wrapper

This is the most complicated approach, but has the potential to be the most performant. What we
will do here is write a Rust function to perform our computation and then expose that function to
python. I know of two use cases where I would recommend this approach. The first is the case when
the pyarrow compute functions are insufficient for your needs. Perhaps your code is too complex or
could be greatly simplified if you pulled in some outside dependency. The second use case is when
you have written a UDF that you’re sharing across multiple projects and have hardened the approach.
It is possible that you can implement your function in rust to give a speed improvement and then
every project that is using this shared UDF will benefit from those updates.

When deciding to use this approach, it’s worth considering how much you think you’ll actually
benefit from the rust implementation to decide if it’s worth the additional effort to maintain and
deploy the python wheels you generate. It is certainly not necessary for every use case.

Due to the excellent work by the python arrow team, we can simplify our work to needing only two
dependencies on the rust side, [arrow-rs](https://github.com/apache/arrow-rs) and
[pyo3](https://pyo3.rs/). I have posted a [minimal example](https://github.com/timsaucer/tuple_filter_example).
You’ll need [maturin](https://github.com/PyO3/maturin) to build the project, and I recommend using
release mode when building.

```bash
maturin develop --release
```

When you write your UDF in rust you generally will need to take these steps

1. Write a function description that takes in some number of python generic objects.
2. Convert these objects to Arrow Arrays of the appropriate type(s).
3. Perform your computation and create a resultant Array.
3. Convert the array into a python generic object.

For the conversion to and from python objects, we can take advantage of the
`ArrayData::from_pyarrow_bound` and `ArrayData::to_pyarrow` functions.  All that remains is to
perform your computation.

We are going to demonstrate doing this computation in two ways. The first is to mimic what we’ve
done in the above approach using pyarrow. In the second we demonstrate iterating through the three
arrays ourselves.

In our first approach, we can expect the performance to be nearly identical to when we used the
pyarrow compute functions. On the rust side we will have slightly less overhead but the heavy
lifting portions of the code are essentially the same between this rust implementation and the
pyarrow approach above.

The reason for demonstrating this, even though it doesn’t provide a significant speedup over
python, is to primarily demonstrate how to make the python to rust with python wrapper
transition. In the second implementation you can see how we can iterate through all of the arrays
ourselves.

```rust
#[pyfunction]
pub fn tuple_filter_fn(
    py: Python<'_>,
    partkey_expr: &Bound<'_, PyAny>,
    suppkey_expr: &Bound<'_, PyAny>,
    returnflag_expr: &Bound<'_, PyAny>,
) -> PyResult<Py<PyAny>> {
    let partkey_arr: PrimitiveArray<Int64Type> =
        ArrayData::from_pyarrow_bound(partkey_expr)?.into();
    let suppkey_arr: PrimitiveArray<Int64Type> =
        ArrayData::from_pyarrow_bound(suppkey_expr)?.into();
    let returnflag_arr: StringArray = ArrayData::from_pyarrow_bound(returnflag_expr)?.into();

    let values_of_interest = vec![
        (1530, 4031, "N".to_string()),
        (6530, 1531, "N".to_string()),
        (5618, 619, "N".to_string()),
        (8118, 8119, "N".to_string()),
    ];

    let mut res: Option<BooleanArray> = None;

    for (partkey, suppkey, returnflag) in &values_of_interest {
        let filtered_partkey_arr = BooleanArray::from_unary(&partkey_arr, |p| p == *partkey);
        let filtered_suppkey_arr = BooleanArray::from_unary(&suppkey_arr, |s| s == *suppkey);
        let filtered_returnflag_arr =
            BooleanArray::from_unary(&returnflag_arr, |s| s == returnflag);

        let part_and_supp = compute::and(&filtered_partkey_arr, &filtered_suppkey_arr)
            .map_err(|e| PyValueError::new_err(e.to_string()))?;
        let resultant_arr = compute::and(&part_and_supp, &filtered_returnflag_arr)
            .map_err(|e| PyValueError::new_err(e.to_string()))?;

        res = match res {
            Some(r) => compute::or(&r, &resultant_arr).ok(),
            None => Some(resultant_arr),
        };
    }

    res.unwrap().into_data().to_pyarrow(py)
}


#[pymodule]
fn tuple_filter_example(module: &Bound<'_, PyModule>) -> PyResult<()> {
    module.add_function(wrap_pyfunction!(tuple_filter_fn, module)?)?;
    Ok(())
}
```

To use this we use the `udf` function in datafusion-python just as before.

```python
from datafusion import udf
import pyarrow as pa
from tuple_filter_example import tuple_filter_fn

udf_using_custom_rust_fn = udf(
    tuple_filter_fn,
    [pa.int64(), pa.int64(), pa.utf8()],
    pa.bool_(),
    "stable",
)
```

That's it! We've now got a third party rust UDF with python wrappers working with DataFusion's
python bindings!

### Rust UDF with initialization

Looking at the code above, you can see that it is hard coding the values we're interested in. There
are many types of UDFs that don't require any additional data provided to them before they start
the computation. The code above is sloppy, so let's clean it up.

We want to write the function to take some additional data. A limitation of the UDFs we create is
that they expect to operate on entire arrays of data at a time. We can get around this problem by
creating an initializer for our UDF. We do this by defining a rust struct that contains the data we
need and implement two methods on this struct, `new` and `__call__`. By doing this we will create a
python object that is callable, so it can be the function we provide to `udf`.

```rust
#[pyclass]
pub struct TupleFilterClass {
    values_of_interest: Vec<(i64, i64, String)>,
}

#[pymethods]
impl TupleFilterClass {
    #[new]
    fn new(values_of_interest: Vec<(i64, i64, String)>) -> Self {
        Self {
            values_of_interest,
        }
    }

    fn __call__(
        &self,
        py: Python<'_>,
        partkey_expr: &Bound<'_, PyAny>,
        suppkey_expr: &Bound<'_, PyAny>,
        returnflag_expr: &Bound<'_, PyAny>,
    ) -> PyResult<Py<PyAny>> {
        let partkey_arr: PrimitiveArray<Int64Type> =
            ArrayData::from_pyarrow_bound(partkey_expr)?.into();
        let suppkey_arr: PrimitiveArray<Int64Type> =
            ArrayData::from_pyarrow_bound(suppkey_expr)?.into();
        let returnflag_arr: StringArray = ArrayData::from_pyarrow_bound(returnflag_expr)?.into();

        let mut res: Option<BooleanArray> = None;

        for (partkey, suppkey, returnflag) in &self.values_of_interest {
            let filtered_partkey_arr = BooleanArray::from_unary(&partkey_arr, |p| p == *partkey);
            let filtered_suppkey_arr = BooleanArray::from_unary(&suppkey_arr, |s| s == *suppkey);
            let filtered_returnflag_arr =
                BooleanArray::from_unary(&returnflag_arr, |s| s == returnflag);

            let part_and_supp = compute::and(&filtered_partkey_arr, &filtered_suppkey_arr)
                .map_err(|e| PyValueError::new_err(e.to_string()))?;
            let resultant_arr = compute::and(&part_and_supp, &filtered_returnflag_arr)
                .map_err(|e| PyValueError::new_err(e.to_string()))?;

            res = match res {
                Some(r) => compute::or(&r, &resultant_arr).ok(),
                None => Some(resultant_arr),
            };
        }

        res.unwrap().into_data().to_pyarrow(py)
    }
}

#[pymodule]
fn tuple_filter_example(module: &Bound<'_, PyModule>) -> PyResult<()> {
    module.add_class::<TupleFilterClass>()?;
    Ok(())
}
```

When you write this, you don't have to call your constructor `new`. The more important part is that
you have `#[new]` designated on the function. With this you can provide any kinds of data you need
during processing. Using this initializer in python is fairly straightforward.

```python
from datafusion import udf
import pyarrow as pa
from tuple_filter_example import TupleFilterClass

tuple_filter_class = TupleFilterClass(values_of_interest)

udf_using_custom_rust_fn_with_data = udf(
    tuple_filter_class,
    [pa.int64(), pa.int64(), pa.utf8()],
    pa.bool_(),
    "stable",
    name="tuple_filter_with_data"
)
```

When you use this approach you will need to provide a `name` argument to `udf`. This is because our
class/struct does not get the `__qualname__` attribute that the `udf` function is looking for. You
can give this udf any name you choose.

### Rust UDF with direct iteration

The final version of our scalar UDF is one where we implement it in rust and iterate through all of
the arrays ourselves. If you are iterating through more than 3 arrays at a time I recommend looking
at [izip](https://docs.rs/itertools/latest/itertools/macro.izip.html) in the
[itertools crate](https://crates.io/crates/itertools). For ease of understanding and since we only
have 3 arrays here I will just explicitly create my own tuple here.

```rust
#[pyclass]
pub struct TupleFilterDirectIterationClass {
    values_of_interest: Vec<(i64, i64, String)>,
}

#[pymethods]
impl TupleFilterDirectIterationClass {
    #[new]
    fn new(values_of_interest: Vec<(i64, i64, String)>) -> Self {
        Self { values_of_interest }
    }

    fn __call__(
        &self,
        py: Python<'_>,
        partkey_expr: &Bound<'_, PyAny>,
        suppkey_expr: &Bound<'_, PyAny>,
        returnflag_expr: &Bound<'_, PyAny>,
    ) -> PyResult<Py<PyAny>> {
        let partkey_arr: PrimitiveArray<Int64Type> =
            ArrayData::from_pyarrow_bound(partkey_expr)?.into();
        let suppkey_arr: PrimitiveArray<Int64Type> =
            ArrayData::from_pyarrow_bound(suppkey_expr)?.into();
        let returnflag_arr: StringArray = ArrayData::from_pyarrow_bound(returnflag_expr)?.into();

        let values_to_search: Vec<(&i64, &i64, &str)> = (&self.values_of_interest)
            .iter()
            .map(|(a, b, c)| (a, b, c.as_str()))
            .collect();

        let values = partkey_arr
            .values()
            .iter()
            .zip(suppkey_arr.values().iter())
            .zip(returnflag_arr.iter())
            .map(|((a, b), c)| (a, b, c.unwrap_or_default()))
            .map(|v| values_to_search.contains(&v));

        let res: BooleanArray = BooleanBuffer::from_iter(values).into();

        res.into_data().to_pyarrow(py)
    }
}
```

We convert the `values_of_interest` into a vector of borrowed types so that we can do a fast search
without creating additional memory. The other option is to turn the `returnflag` into a `String`
but that memory allocation is unnecessary. After taht we use two `zip` operations so that we can
iterate over all three columns in a single pass. Since each `zip` will return a tuple of two
elements, a quick `map` turns them into the tuple format we need. Also, `StringArray` is a little
different in the buffer it uses, so it is treated slightly differently from the others.

## User Defined Aggregate Function

Writing a user defined aggregate function or user defined window function is slightly more complex
than scalar functions. This is because we must accumulate values and there is no guarantee that one
batch will contain off the values we are aggregating over. For this we need to define an
`Accumulator` which will do a few things.

- Process a batch and compute an internal state
- Share the state so that we can combine multiple batches
- Merge the results across multiple batches
- Return the final result

In the example below, we're going to look at customer orders and we want to know per customer ID,
how much they have ordered total. We want to ignore small orders, which we define as anything over
5000.

```python
from datafusion import Accumulator, udaf
import pyarrow as pa
import pyarrow.compute as pc

IGNORE_THESHOLD = 5000.0
class AboveThersholdAccum(Accumulator):
    def __init__(self) -> None:
        self._sum = 0.0

    def update(self, values: pa.Array) -> None:
        over_threshold = pc.greater(values, pa.scalar(IGNORE_THESHOLD))
        sum_above = pc.sum(values.filter(over_threshold)).as_py()
        if sum_above is None:
            sum_above = 0.0
        self._sum = self._sum + sum_above

    def merge(self, states: List[pa.Array]) -> None:
        self._sum = self._sum + pc.sum(states[0]).as_py()

    def state(self) -> List[pa.Scalar]:
        return [pa.scalar(self._sum)]

    def evaluate(self) -> pa.Scalar:
        return pa.scalar(self._sum)

sum_above_threshold = udaf(AboveThersholdAccum, [pa.float64()], pa.float64(), [pa.float64()], 'stable')

df_orders.aggregate([col("o_custkey")],[sum_above_threshold(col("o_totalprice")).alias("sales")]).show()
```

Since we are doing a `sum` we can keep a single value as our internal state. When we call `update()`
we will process a single array and update the internal state, which we share with the `state()`
function. For larger batches we may `merge()` these states. It is important to note that the
`states` in the `merge()` function are an array of the values returned from `state()`. It is
entirely possible that the `merge` function is significantly different than the `update`, though in
our example they are very similar.

## Performance Comparison

For the scalar functions above, we performed a timing evaluation, repeating the operation 100
times. For this simple example these are our results.

```
+-----------------------------+--------------+---------+
| approach                    | Average Time | Std Dev |
+-----------------------------+--------------+---------+
| python udf                  | 4.969        | 0.062   |
| simple filter               | 1.075        | 0.022   |
| explicit filter             | 0.685        | 0.063   |
| pyarrow compute             | 0.529        | 0.017   |
| arrow rust compute          | 0.511        | 0.034   |
| arrow rust compute as class | 0.502        | 0.011   |
| rust custom iterator        | 0.478        | 0.009   |
+-----------------------------+--------------+---------+
```

As expected, the conversion to python objects is by far the worst performance. As soon as we drop
into using any functions that keep the data entirely on the rust side we see a near 10x speed
improvement. Then as we increase our complexity from using pyarrow compute functions to
implementing the UDF in rust we see incremental improvements. Our fastest approach - iterating
through the arrays ourselves does operate nearly 10% faster than the pyarrow compute approach.

## Final Thoughts and Recommendations

For anyone who is curious about [DataFusion](https://datafusion.apache.org/) I highly recommend
giving it a try. This post was designed to make it easier for new users to the python implementation
to work with User Defined Functions by giving a few examples of how one might implement these.

When it comes to designing UDFs, I strongly recommend seeing if you can write your UDF using
[pyarrow functions](https://arrow.apache.org/docs/python/api/compute.html) rather than pure python
objects. These will give you enormous speed benefits. If you must do something that isn't well
represented by the pyarrow compute functions, then I would consider using a Rust based UDF in the
manner shown above.

Lastly, the Apache Arrow and DataFusion community is an active group of very helpful people working
to make a great tool. If you want to get involved, please take a look at the
[online documenation](https://datafusion.apache.org/python/) and jump in to help with one of the
(open issues)[https://github.com/apache/datafusion-python/issues].