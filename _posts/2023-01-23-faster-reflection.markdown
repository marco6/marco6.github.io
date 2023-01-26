---
layout: post
title:  Golang reflection, a faster route?
date:   2023-01-22 18:45:00 +0100
tags: reflection golang
---
    
<link rel="stylesheet" href="/assets/css/custom.css">

Reflection can be very handy and makes it possible to achieve flexibility. One of its typical use-cases is serialization.

Let's write the simplest example I can think of. This serializer will write:
 - byte, \[u\]int8 and bools as a single byte;
 - bigger integers as Varints ([leb-128](https://en.wikipedia.org/wiki/LEB128));
 - arrays as the concatenation of its content;
 - strings and slices as len + content;
 - structs as its fields, without any structure; 
 - pointers as a bool + contents;
 - maps as len + key-value stream;
 - interfaces won't be supported for now.

Let's capture the basics of the writing in an interface:

```golang
type BinaryWriter interface {
    WriteBool(bool)
    WriteInt8(int8)
    WriteUint8(uint8)
    WriteInt(int64)
    WriteUint(uint64)
    WriteFloat32(float32)
    WriteFloat64(float64)
    WriteString(string)
}
```

The serializer is then just using reflection to route each time to the right method:[^1]:

[^1]: Please, note that I'm willingly ignoring errors as this code is not the point of the article. Also the format is so dumb it's no surprise this code is useless.


```golang
package fast_reflection

import (
    "fmt"
    "reflect"
)

func Serialize(writer BinaryWriter, val any) error {
    if val == nil {
        return fmt.Errorf("can't serialize nil")
    }

    value := reflect.ValueOf(val)

    if value.Kind() != reflect.Pointer {
        return fmt.Errorf("val must be a pointer")
    }

    return serializeValue(writer, value.Elem())
}

func serializeValue(writer BinaryWriter, value reflect.Value) error {
    switch value.Kind() {
    case reflect.Bool:
        writer.WriteBool(value.Bool())

    case reflect.Int8:
        writer.WriteInt8(int8(value.Int()))

    case reflect.Uint8:
        writer.WriteUint8(uint8(value.Uint()))

    case reflect.Int, reflect.Int16, reflect.Int32, reflect.Int64:
        writer.WriteInt(value.Int())

    case reflect.Uint, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        writer.WriteUint(value.Uint())

    case reflect.Float32:
        writer.WriteFloat32(float32(value.Float()))

    case reflect.Float64:
        writer.WriteFloat64(value.Float())

    case reflect.Complex64:
        cpx := value.Complex()
        writer.WriteFloat32(float32(real(cpx)))
        writer.WriteFloat32(float32(imag(cpx)))

    case reflect.Complex128:
        cpx := value.Complex()
        writer.WriteFloat64(real(cpx))
        writer.WriteFloat64(imag(cpx))

    case reflect.Array:
        len := value.Len()
        for i := 0; i < len; i++ {
            serializeValue(writer, value.Index(i))
        }

    case reflect.Slice:
        len := value.Len()
        writer.WriteInt(int64(len))
        for i := 0; i < len; i++ {
            serializeValue(writer, value.Index(i))
        }

    case reflect.String:
        writer.WriteString(value.String())

    case reflect.Map:
        if value.IsNil() {
            writer.WriteInt(0)
        } else {
            writer.WriteInt(int64(value.Len()))
            for iter := value.MapRange(); iter.Next(); {
                serializeValue(writer, iter.Key())
                serializeValue(writer, iter.Value())
            }
        }

    case reflect.Pointer:
        if value.IsNil() {
            writer.WriteBool(false)
        } else {
            writer.WriteBool(true)
            serializeValue(writer, value.Elem())
        }

    case reflect.Struct:
        // Let's ignore the complexity deriving from exported vs non-exported fields
        for i := 0; i < value.NumField(); i++ {
            serializeValue(writer, value.Field(i))
        }

    case reflect.Uintptr, reflect.Chan, reflect.Func, reflect.Interface, reflect.UnsafePointer:
        return fmt.Errorf("Unsupported kind %s", value.Kind())
    }

    return nil
}
```

A bit longer than I wanted, but simple to read (and sure enough not suitable for production!).

There's some weirdness though. For example, you must pass a pointer, but the **public interface** of the method doesn't seem to be aware of that, so:

```golang
var value MyAwesomeType

fast_reflection.Serialize(reader, value) // run-time error!
```

If you are lucky enough to use go1.18+, then you can use the new shiny Generics to work around this akwardness:

```golang
func Serialize[T any](writer BinaryWriter, val *T) error {
    if val == nil {
        return fmt.Errorf("can't serialize nil")
    }

    value := reflect.ValueOf(val)

    // Don't need to check if it's a pointer anymore

    return serializeValue(writer, value.Elem())
}
```

## How much does it costs?

Evaluating a technology or a solution needs to take into account the cost. In this case we are not talking about cash, but execution speed.

So, let's write a benchmark for that, as a baseline.

I will use a simple, yet representative, struct:

```golang
type simpleStruct struct {
    I int
    U uint
    S string
    L []int
    P *int
    M map[int]int
}
```

And a very naive hand-written serialization function:

```golang
func serializeSimpleStruct(writer io.Writer, s simpleStruct) error {
    writer.WriteInt(int64(s.I))
    writer.WriteUint(uint64(s.U))
    writer.WriteString(s.S)

    writer.WriteInt(int64(len(s.L)))
    for _, item := range s.L {
        writer.WriteInt(int64(item))
    }

    if s.P == nil {
        writer.WriteBool(false)
    } else {
        writer.WriteBool(true)
        writer.WriteInt(int64(*s.P))
    }

    if s.M == nil {
        writer.WriteInt(0)
    } else {
        writer.WriteInt(int64(len(s.M)))

        for k, v := range s.M {
            writer.WriteInt(int64(k))
            writer.WriteInt(int64(v))
        }
    }

    return nil
}
```

To banchmark our reflection code, it is necessary to remove as much as possible things not related to the access method. For this, we can write a very simple `NilBinaryWriter` which will basically do nothing. It's also useful to mark all its methods as `go:noinline`, otherwise the hand-written solution would probably be able to optimize too much, making the comparrison unfair.

```golang
type NilBinaryWriter struct{}

var _ BinaryWriter = &NilBinaryWriter{}

//go:noinline
func (*NilBinaryWriter) WriteBool(bool) {}

//go:noinline
func (*NilBinaryWriter) WriteFloat32(float32) {}

//go:noinline
func (*NilBinaryWriter) WriteFloat64(float64) {}

//go:noinline
func (*NilBinaryWriter) WriteInt(int64) {}

//go:noinline
func (*NilBinaryWriter) WriteInt8(int8) {}

//go:noinline
func (*NilBinaryWriter) WriteString(string) {}

//go:noinline
func (*NilBinaryWriter) WriteUint(uint64) {}

//go:noinline
func (*NilBinaryWriter) WriteUint8(uint8) {}
```

The benchmark is then straightforward:

```golang
func BenchmarkSerialize(b *testing.B) {
    s := simpleStruct{
        I: 1,
        U: 2,
        S: "3",
        L: []int{4, 5},
        P: new(int),
        M: map[int]int{6: 7, 8: 9},
    }

    b.Run("hand-written", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            writer := &bytes.Buffer{}
            serializeSimpleStruct(writer, s)
        }

    })

    b.Run("reflect", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            writer := &bytes.Buffer{}
            fast_reflection.Serialize(writer, s)
        }
    })
}
```

Which gives:

```plain
> go test . -bench=. -benchmem
goos: linux
goarch: amd64
pkg: fast_reflection
cpu: Intel(R) Core(TM) i5-6300U CPU @ 2.40GHz
BenchmarkSerialize/hand-written-4                8458156               130.5 ns/op             0 B/op          0 allocs/op
BenchmarkSerialize/reflect-4                     2211645               515.6 ns/op            32 B/op          4 allocs/op
```

So, from this we can see:

 - reflection is around 4x slower;
 - reflection allocates while the hand written one doesn't.


## In search for faster route

Optimization is about compromise. In the benchmark the trade-off is clear: to gain performance we are sacrificing maintainability [^2] (by hand-writing the serialization code).

[^2]: this is usually a **very bad idea&#x2122;**

Of course, one could use tools like `go generate` to remove the "hand written" part. While this is viable, that's not what this article is about.

Let's introduce the `unsafe` package. As the name implies, this package can be used to defy the normal rules and if not used carefully can result in a malformed program. There are, however, usage patterns that are accepted and considered *safe* (a safe usage of an `unsafe` feature...). Quoting the [documentation](https://pkg.go.dev/unsafe#Pointer) about `unsafe.Pointer`:

```
The following patterns involving Pointer are valid. Code not using these patterns is likely to be invalid today or to become invalid in the future. Even the valid patterns below come with important caveats. 

(1) Conversion of a *T1 to Pointer to *T2.

Provided that T2 is no larger than T1 and that the two share an equivalent memory layout, this conversion allows reinterpreting data of one type as data of another type.

...

(3) Conversion of a Pointer to a uintptr and back, with arithmetic. 

If p points into an allocated object, it can be advanced through the object by conversion to uintptr, addition of an offset, and conversion back to Pointer. 

    p = unsafe.Pointer(uintptr(p) + offset)

The most common use of this pattern is to access fields in a struct or elements of an array: 

    // equivalent to f := unsafe.Pointer(&s.f)
    f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))

    // equivalent to e := unsafe.Pointer(&x[i])
    e := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i*unsafe.Sizeof(x[0]))

It is valid both to add and to subtract offsets from a pointer in this way. It is also valid to use &^ to round pointers, usually for alignment. In all cases, the result must continue to point into the original allocated object. 
```

Interesting. Go is saying that you can use **pointer arithmetics** to:

 - access array elements
 - access struct fields

as long as those addresses don't outlive the object itself (clearly).

This means that we can compose a function to perform serialization/deserialization. Something like:

```golang
func MakeSerializer(reflect.Type) func(BinaryWriter, any) error
```

Sinc this function must deal with the "non-unsafe" world and we are using "unsafe" version inside, it is better to demand the real composition of the writer to a non-exported function and wrap it like:

```golang
func makeSerializer(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error)

func MakeSerializer(typ reflect.Type) (func(BinaryWriter, any) error, error) {

    serializer, err := makeUsafeWriter(typ)

    if err != nil {
        return nil, err
    }

    return func(writer io.Writer, val any) error {
        if val == nil {
            return fmt.Errorf("can't serialize nil")
        }

        value := reflect.ValueOf(val)

        if value.Kind() != reflect.Pointer {
            return fmt.Errorf("val must be a pointer")
        }

        serializer(writer, value.UnsafePointer())

        return nil
    }, nil
}
```

Just like in the other case, it is possible to use generics to improve the usage pattern:

```golang
func MakeSerializer[T any]() (func(BinaryWriter, *T) error, error) {
    var _t T
    serializer, err := makeSerializer(reflect.TypeOf(_t))

    if err != nil {
        return nil, err
    }

    return func(writer BinaryWriter, val *T) error {
        if val == nil {
            return fmt.Errorf("can't serialize nil")
        }

        serializer(writer, unsafe.Pointer(val))

        return nil
    }, nil
}
```

It is convenient first to define some helpers. Most of them are as trivial as casting the pointer to the right type and calling the right `writeXXXUnsafe` function. 

```golang
func writeBoolUnsafe(writer BinaryWriter, pVal unsafe.Pointer) {
    value := *(*bool)(pVal)
    writer.WriteBool(value)
}
```

This way, `makeUsafeWriter` will just return the right function for the type at hand, eventually composing them for aggregates. Something like:

```golang
func makeUsafeWriter(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error) {
    switch typ.Kind() {
    case reflect.Bool:
        return writeBoolUnsafe, nil

    case reflect.Int8:
        return writeInt8Unsafe, nil

    case reflect.Int16:
        return writeInt16Unsafe, nil
    ...
    case reflect.Uintptr, reflect.Chan, reflect.Func, reflect.Interface, reflect.UnsafePointer:
        return nil, fmt.Errorf("Unsupported kind %s", typ.Kind())
    }

    panic("can't be here")
}
```

Arrays, slices, struct, pointers and maps are trickier.

For arrays we will need to capture the length with a 2nd order function:

```golang
func makeArrayUnsafeWriter(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error) {
    len := typ.Len()
    elemTyp := typ.Elem()
    elemSize := elemTyp.Size()
    elemSerializer, err := makeUsafeWriter(elemTyp)

    if err != nil {
        return nil, err
    }

    return func(writer io.Writer, pVal unsafe.Pointer) {
        for i := 0; i < len; i++ {
            pItem := unsafe.Pointer(uintptr(pVal) + uintptr(i)*elemSize) // This is accepted!
            elemSerializer(writer, pItem)
        }
    }, nil
}
```

For slices there's a bit more of work as the length is not know at "composition time".

```golang
func makeSliceUnsafeWriter(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error) {
    // Len is fixed!
    elemTyp := typ.Elem()
    elemSize := elemTyp.Size()
    elemSerializer, err := makeSerializer(elemTyp)

    if err != nil {
        return nil, err
    }

    return func(writer BinaryWriter, pVal unsafe.Pointer) {
        // in go1.20 we will have `SliceData` for this...
        slice := (*reflect.SliceHeader)(pVal)

        writer.WriteInt(int64(slice.Len))

        for i := 0; i < slice.Len; i++ {
            pItem := unsafe.Pointer(slice.Data + uintptr(i)*elemSize)
            elemSerializer(writer, pItem)
        }
    }, nil
}
```

That conversion is considered by Go's linter as unsafe (e.g. not part of that "good use" list). It seems to me that `slice.Data` points to the slice's backing array, and arrays can be used this way, it is logical to believe that this is ok (and that that warning is a false positive).

Struct can be analyzed beforehand, saving a list of functions to call each time:

```golang
func makeStructUnsafeWriter(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error) {
    fields := make([]func(BinaryWriter, unsafe.Pointer), typ.NumField())

    for i := 0; i < typ.NumField(); i++ {
        field := typ.Field(i)
        fieldOffset := field.Offset
        fieldWriter, err := makeSerializer(field.Type)
        if err != nil {
            return nil, err
        }

        fields[i] = func(writer BinaryWriter, pVal unsafe.Pointer) {
            fieldWriter(writer, unsafe.Pointer(uintptr(pVal)+fieldOffset))
        }
    }

    return func(writer BinaryWriter, pVal unsafe.Pointer) {
        for _, field := range fields {
            field(writer, pVal)
        }
    }, nil
}
```

Now, pointers can be dereferenced by converting the pointer to `*unsafe.Pointer` as the rule (1) from `unsafe.Pointer` says that I can convert between types that share the same memory layout.

```golang
func makePointerUnsafeWriter(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error) {
    elemWriter, err := makeSerializer(typ.Elem())

    if err != nil {
        return nil, err
    }

    return func(writer BinaryWriter, pVal unsafe.Pointer) {
        value := *(*unsafe.Pointer)(pVal)
        if value == nil {
            writer.WriteBool(false)
        } else {
            writer.WriteBool(true)
            elemWriter(writer, value)
        }
    }, nil
}
```

Last, maps. Unfortunately, for maps, there is simply no way to iterate using `unsafe.Pointer`s. If you used Go for long enough, it is probably clear to you that maps are a somewhat 2nd-class citizen in Go. I might write an article in the future on why.

So, how can we write something working? Well... We could go back to reflection! Thanks to `reflect.NewAt` we can get a value from an existing pointer (given that we know the type).

Let's try:

```golang
func makeMapUnsafeWriter(typ reflect.Type) (func(io.Writer, unsafe.Pointer), error) {
    keyWriter, err := makeUsafeWriter(typ.Key())
    if err != nil {
        return nil, err
    }

    elemWriter, err := makeUsafeWriter(typ.Elem())
    if err != nil {
        return nil, err
    }

    return func(writer io.Writer, pVal unsafe.Pointer) {
        if pVal == nil {
            writeInt(writer, 0)
        } else {
            value := reflect.NewAt(typ, pVal).Elem()

            writeInt(writer, int64(value.Len()))

            for iter := value.MapRange(); iter.Next(); {
                keyWriter(writer, iter.Key().UnsafePointer())
                elemWriter(writer, iter.Value().UnsafePointer())
            }
        }
    }, nil
}
```

Seems neat! Except that it crashes &#x1F4A5; 

The problem lies in the `iter.Key()` and `iter.Value()` which are both **non addressable**. This is the contrived way of Go to solve the lack of `const` (or equivalent) functionality (e.g. you cannot take the address of a const value otherwise you could be changing it... Except for [strings](https://pkg.go.dev/unsafe@go1.20rc3#StringData) &#x1F615; ). 

If we want to stay within the "safe" usage of the `unsafe` package, the only way around that is to copy those values outside. This will cost few cicles and makes the inner loop ugly:

```golang
key := reflect.New(typ.Key())
elem := reflect.New(typ.Elem())

for iter := value.MapRange(); iter.Next(); {
    key.Elem().Set(iter.Key())
    elem.Elem().Set(iter.Value())
    keyWriter(writer, key.UnsafePointer())
    elemWriter(writer, elem.UnsafePointer())
}
```

So, let's benchmark what we have.

```plain
goos: linux
goarch: amd64
pkg: fast_reflection
cpu: Intel(R) Core(TM) i5-6300U CPU @ 2.40GHz
BenchmarkSerialize/hand-written-4                8856024               115.1 ns/op             0 B/op          0 allocs/op
BenchmarkSerialize/reflect-4                     2778475               496.1 ns/op            32 B/op          4 allocs/op
BenchmarkSerialize/unsafe-4                      1998032               599.4 ns/op            48 B/op          6 allocs/op
```

Wait what? It became slower! We can now run to our manager saying that in a day of work we managed to:

 - make the code much less readable;
 - make it slower;
 - make it consume more memory.

Or we can understand where the problem is.

First of all, that strange `map` code is making copies at each step of the loop. Can we make that better? For now, we can use the "normal" reflect code so that it's at least in par with the rest of the code.

```golang
func makeMapUnsafeWriter(typ reflect.Type) (func(BinaryWriter, unsafe.Pointer), error) {
    return func(writer BinaryWriter, pVal unsafe.Pointer) {
        if pVal == nil {
            writer.WriteInt(0)
        } else {
            m := reflect.NewAt(typ, pVal).Elem()

            writer.WriteInt(int64(m.Len()))
            for iter := m.MapRange(); iter.Next(); {
                serializeValue(writer, iter.Key())
                serializeValue(writer, iter.Value())
            }
        }
    }, nil
}
```

And another round of tests gives:

```plain
goos: linux
goarch: amd64
pkg: fast_reflection
cpu: Intel(R) Core(TM) i5-6300U CPU @ 2.40GHz
BenchmarkSerialize/hand-written-4               12570352                99.26 ns/op            0 B/op          0 allocs/op
BenchmarkSerialize/reflect-4                     2666583               448.8 ns/op            32 B/op          4 allocs/op
BenchmarkSerialize/unsafe-4                      2668941               422.2 ns/op            32 B/op          4 allocs/op
```

Oh yeah! It's this 5% that will get your salary up!

No, honestly, the result is somewhat better, but clearly not the way it's intended. We are supposedly doing **less** work (as we are not inspecting the structure of our data, but just acting on it), but the result might very well some testing fluctation. 

What's happening? Is combining lambdas this costly? Maybe!

First of all, as far as I know, go (as of v1.20) can't inline closures properly, not even [not-escaping ones](https://github.com/golang/go/issues/15561). 

But there is another hidden cost of functions. Go have a curious way to represent goroutines, in the sense that it will allocate a very small stack and it will grow on demand. This basically means that each function call will need first to check if the stack is enough for its stack-frame, and, if not, it will resize it. That's some overhead.

It is possible to remove that (at your own risk) with the directive `go:nosplit` [^3].

[^3]: curiously enough, that directive cannot be applied to closures!

```plain
goos: linux
goarch: amd64
pkg: fast_reflection
cpu: Intel(R) Core(TM) i5-6300U CPU @ 2.40GHz
BenchmarkSerialize/hand-written-4               13102244                99.27 ns/op            0 B/op          0 allocs/op
BenchmarkSerialize/reflect-4                     2841808               458.0 ns/op            32 B/op          4 allocs/op
BenchmarkSerialize/unsafe-4                      2621190               409.8 ns/op            32 B/op          4 allocs/op
```

Ok, then the problem really is how bad go is at inlining/calling functions. Also, fluctuations in the result of a microbenchmark like this clearly point to the fact that the benchmarking library in Go needs some serious improvements.

How could we enable more inlining? Can we have a switch (again), but lighter than all that reflection code?

What about writing a very simple interpreter of a rudimental "instruction set"?

## A simple serialization language

Ok, so let's say that we want to make a language to save our decisions about what to serialize. Just like a normal instruction set, we will have some opcode:

```golang
type opcode int
```

Each `opcode` will possibly have arguments inside of the "bytecode". Each `opcode` is a serialization operation.

```golang
const (
    invalid    opcode = iota
    writeBool
    writeInt8
    writeInt16
    writeInt32
    writeInt64
    writeInt
    writeUint8
    writeUint16
    writeUint32
    writeUint64
    writeUint
    writeFloat32
    writeFloat64
    writeComplex64
    writeComplex128
    writeString
    writeSlice
    writeMap 
    writePointer
)
```

Each of them will sure enough need an offset for the field (in case we are not serializing a struct, it will be 0).

The execution loop will be something like:

```golang
func runSerializerCode(writer BinaryWriter, pVal unsafe.Pointer, code []int, mapTypes []reflect.Type) {
    pc := 0 // program counter

    for pc < len(code) {
        op := opcode(code[pc])
        pc++

        offset := uintptr(code[pc])
        pc++

        fieldPtr := unsafe.Pointer(uintptr(pVal) + offset)

        switch op {
        case writeBool:
            writeBoolUnsafe(writer, fieldPtr)
        case writeInt8:
            writeInt8Unsafe(writer, fieldPtr)
        ...
    }
}
```

Turns out that arrays can be inlined easily by unrolling the loop. 

In the case of pointers and slices we need to be able to jump instruction (forward for pointers and backwards for slices).

We can think in terms of "subroutines":

```plain
writePointer <offset> <subroutine code length>
<subroutine code>
...
writeSlice <offset> <item size> <subroutine code length>
<subroutine code>
```

In this way we can call `runSerializerCode` recursively on the subroutine code. Something like:

```golang
case writePointer:
    subroutine := code[pc]
    pc++
    value := *(*unsafe.Pointer)(fieldPtr)
    if value == nil {
        writer.WriteBool(false)
    } else {
        writer.WriteBool(true)
        runSerializerCode(writer, value, code[pc:(pc+subroutine)], mapTypes)
    }
    pc += int(subroutine)
```

Just like before, maps are a bit akward, so we will just call the original routine for now with:

```golang
func writeMapUnsafe(writer BinaryWriter, pVal unsafe.Pointer, typ reflect.Type) {
    if pVal == nil {
        writer.WriteInt(0)
    } else {
        m := reflect.NewAt(typ, pVal).Elem()

        writer.WriteInt(int64(m.Len()))
        for iter := m.MapRange(); iter.Next(); {
            serializeValue(writer, iter.Key())
            serializeValue(writer, iter.Value())
        }
    }
}
```

This will need to have a type. We will save the type of the map in the `mapTypes` argument. Finally, the wrapper (again using generics):

```golang
func MakeSerializer[T any]() (func(BinaryWriter, *T) error, error) {
    code := make([]int, 0, 2)
    typsMap := make(map[reflect.Type]int)

    var _t T
    makeSerializerCode(reflect.TypeOf(_t), 0, &code, typsMap)

    typs := make([]reflect.Type, len(typsMap))

    for k, v := range typsMap {
        typs[v] = k
    }

    return func(writer BinaryWriter, val *T) error {
        if val == nil {
            return fmt.Errorf("can't serialize nil")
        }

        runSerializerCode(writer, unsafe.Pointer(val), code, typs)

        return nil
    }, nil
}
```

Testing all of that gives almost the same results of the last time:

```plain
goos: linux
goarch: amd64
pkg: fast_reflection
cpu: Intel(R) Core(TM) i5-6300U CPU @ 2.40GHz
BenchmarkSerialize/hand-written-4               13660764               100.3 ns/op             0 B/op          0 allocs/op
BenchmarkSerialize/reflect-4                     2661003               470.7 ns/op            32 B/op          4 allocs/op
BenchmarkSerialize/unsafe-4                      2637398               406.2 ns/op            32 B/op          4 allocs/op
```

The big difference there is that I removed all the unsafe `go:nosplit`, so the stack is safe this time.

I feel like this test is leading to a big performance hit becouse we are using `serializeValue` for maps, so let's compare the case where maps aren't there. I will use this struct this time:

```golang
type innerStruct struct {
    a [4]int
    b byte
}

type nestedStruct struct {
    I int
    U uint
    S string
    L []int
    P *int
    N innerStruct
}
```

Aaand the result is:

```plain
goos: linux
goarch: amd64
pkg: fast_reflection
cpu: Intel(R) Core(TM) i5-6300U CPU @ 2.40GHz
BenchmarkNestedStruct/hand-written-4            37589709                33.42 ns/op            0 B/op          0 allocs/op
BenchmarkNestedStruct/reflect-4                  6968936               178.0 ns/op             0 B/op          0 allocs/op
BenchmarkNestedStruct/unsafe-4                  15621717                72.92 ns/op            0 B/op          0 allocs/op
```

OH YEAH! The `reflect` code is consistently 4-5x slower while our new shiny code is 2-2.5x only!

You can find the complete listing [here](https://gist.github.com/marco6/9c63524da9738ce2c61fb7457ff0bc84)

## Wrapping up

So, we discussed how difficult it is to beat "boring" reflection code in Go. We saw how bad it is not to inline closures and that the cost of having a resizable stack is not zero.

Also it is clear, given the length of this article, that I need to come a long way to learn how to write concise, yet useful articles.

<hr>
