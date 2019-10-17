# Hint Conditional

* Proposal: [HXP-NNNN](NNNN-hint-conditional.md)
* Author: [Kim Lindblom](https://github.com/Drakim)

## Introduction

Sometimes we know things the compiler does not, and leaving code such as conditional statements to nudge
compilation in the right direction can have unintended consequences. The hint conditional is an
attempt to transparently utilize the compiler's inner knowledge during the optimization/inline step.

## Motivation

The Haxe compiler can do miracles with optimizations. However, I've often come across situations
where I wish I could help it along. Macros are unable to help me because they execute way before
any compiler optimizations, inlining, and evaluation takes place.

The most straightforward example is when a zero is written to zero initialized memory. I've created
a somewhat realistic example to demonstrate how this unfolds and how it could be improved by this
proposal.

Example:

```haxe
    // Some fun matrix math, which is then stored in a buffer
    class Main {
        static public function main() {
            var dataBuffer = new DataBuffer();
            var matrix1 = Matrix.identity().translation(75.0, 80.0);
            var matrix2 = Matrix.identity().scale(5.0, 3.0);
            var matrix3 = Matrix.identity().translation(30.0, 30.0);
            var projection = Matrix.defaultProjection();
            var result = matrix1.multiply(matrix2).multiply(matrix3).multiply(projection);
            dataBuffer.writeMatrix(result);
        }
    }
    class Matrix {
        // lots of code
    }
    class DataBuffer {
        public var data:js.lib.Float64Array = new js.lib.Float64Array(6);
        public function new() {}
        public inline function writeMatrix(_matrix:Matrix) {
            data[0] = _matrix.a;
            data[1] = _matrix.b;
            data[2] = _matrix.c;
            data[3] = _matrix.d;
            data[4] = _matrix.e;
            data[5] = _matrix.f;
        }
    }
```

Unoptimized output:

```js
    // No inlining
    Main.main = function() {
        var dataBuffer = new DataBuffer();
        var matrix1 = Matrix.identity().translation(75.0, 80.0);
        var matrix2 = Matrix.identity().scale(5.0, 3.0);
        var matrix3 = Matrix.identity().translation(30.0, 30.0);
        var projection = Matrix.defaultProjection();
        var result = matrix1.multiply(matrix2).multiply(matrix3).multiply(projection);
        dataBuffer.writeMatrix(result);
    };
```

Optimized output:

```js
    // Matrix constructor, methods, and DataBuffer.writeMatrix are inlined
    var dataBuffer = new DataBuffer();
    dataBuffer.data[0] = 0.016666666666666666;
    dataBuffer.data[1] = 0.;
    dataBuffer.data[2] = 0.;
    dataBuffer.data[3] = -0.01;
    dataBuffer.data[4] = 220.;
    dataBuffer.data[5] = 173.;
```

The especially great part of this is that if we run into a situation where the
compiler cannot safely assume the mathematical outcome of these matrix operations,
it makes best effort attempt which is still very good. We can see this by introducing
an outside element:

```haxe
    static public function main() {
        var foo = new Foo();
        var dataBuffer = new DataBuffer();
        var matrix1 = Matrix.identity().translation(75.0, 80.0);
        var matrix2 = Matrix.identity().scale(5.0, 3.0);
        var matrix3 = Matrix.identity().translation(30.0, 30.0);
        var projection = Matrix.defaultProjection();
        var result = matrix1.multiply(matrix2).multiply(foo.chaosMatrix).multiply(matrix3).multiply(projection);
        dataBuffer.writeMatrix(result);
    }
```

And as a result the output isn't nearly as neat, but most of the matrices are still optimized away:

```js
    Main.main = function() {
        var dataBuffer = new DataBuffer();
        var _matrix = new Foo().chaosMatrix;
        var _a = _matrix.a * 5. + _matrix.b * 0.;
        var _b = _matrix.a * 0. + _matrix.b * 3.;
        var _c = _matrix.c * 5. + _matrix.d * 0.;
        var _d = _matrix.c * 0. + _matrix.d * 3.;
        var _a1 = _a + 0. * _c;
        var _b1 = _b + 0. * _d;
        var _c1 = 0. * _a + _c;
        var _d1 = 0. * _b + _d;
        dataBuffer.data[0] = 0.0033333333333333335 * _a1 + 0. * _c1;
        dataBuffer.data[1] = 0.0033333333333333335 * _b1 + 0. * _d1;
        dataBuffer.data[2] = 0. * _a1 + -0.0033333333333333335 * _c1;
        dataBuffer.data[3] = 0. * _b1 + -0.0033333333333333335 * _d1;
        dataBuffer.data[4] = -1. * _a1 + _c1 + (30. * _a + 30. * _c + (_matrix.e * 5. + _matrix.f * 0. + 75.));
        dataBuffer.data[5] = -1. * _b1 + _d1 + (30. * _b + 30. * _d + (_matrix.e * 0. + _matrix.f * 3. + 80.));
    };
```

But let us introduce another optimization. Our DataBuffer class is by default
zero-initialized, meaning writing the value 0 to it does absolutely nothing.
We can add some conditionals to help the compiler along:

```haxe
    class DataBuffer {
        public var data:js.lib.Float64Array = new js.lib.Float64Array(6);
        public function new() {}
        public inline function writeMatrix(_matrix:Matrix) {
            if(_matrix.a != 0) data[0] = _matrix.a;
            if(_matrix.b != 0) data[1] = _matrix.b;
            if(_matrix.c != 0) data[2] = _matrix.c;
            if(_matrix.d != 0) data[3] = _matrix.d;
            if(_matrix.e != 0) data[4] = _matrix.e;
            if(_matrix.f != 0) data[5] = _matrix.f;
        }
    }
```

Our first example is now even more optimized, two out of the six write operations have been eliminated, completely free of charge:

```js
    Main.main = function() {
        var dataBuffer = new DataBuffer();
        dataBuffer.data[0] = 0.016666666666666666;
        dataBuffer.data[3] = -0.01;
        dataBuffer.data[4] = 220.;
        dataBuffer.data[5] = 173.;
    };
```

Unfortunately, our second example didn't quite benefit as much:

```js
    Main.main = function() {
        var dataBuffer = new DataBuffer();
        var _matrix = new Foo().chaosMatrix;
        var _a = _matrix.a * 5. + _matrix.b * 0.;
        var _b = _matrix.a * 0. + _matrix.b * 3.;
        var _c = _matrix.c * 5. + _matrix.d * 0.;
        var _d = _matrix.c * 0. + _matrix.d * 3.;
        var _a1 = _a + 0. * _c;
        var _b1 = _b + 0. * _d;
        var _c1 = 0. * _a + _c;
        var _d1 = 0. * _b + _d;
        var _a2 = 0.0033333333333333335 * _a1 + 0. * _c1;
        var _b2 = 0.0033333333333333335 * _b1 + 0. * _d1;
        var _c2 = 0. * _a1 + -0.0033333333333333335 * _c1;
        var _d2 = 0. * _b1 + -0.0033333333333333335 * _d1;
        var _e = -1. * _a1 + _c1 + (30. * _a + 30. * _c + (_matrix.e * 5. + _matrix.f * 0. + 75.));
        var _f = -1. * _b1 + _d1 + (30. * _b + 30. * _d + (_matrix.e * 0. + _matrix.f * 3. + 80.));
        if(_a2 != 0) { dataBuffer.data[0] = _a2; }
        if(_b2 != 0) { dataBuffer.data[1] = _b2; }
        if(_c2 != 0) { dataBuffer.data[2] = _c2; }
        if(_d2 != 0) { dataBuffer.data[3] = _d2; }
        if(_e != 0) { dataBuffer.data[4] = _e; }
        if(_f != 0) { dataBuffer.data[5] = _f; }
    };
```

Since we introduced uncertainty, the compiler doesn't know if the resulting
matrix values end up being zero or not, so it doesn't dare eliminate any of
the buffer write operations. However, even worse, it has left behind our little
conditionals, which actually ends up slowing our code. It's much faster to simply
write an unneccessary zero to the already zeroed buffer than to conditionally check
before writing.

This is not solvable with Haxe macros, because they execute before the optimizing
and inlining step of the compiler, meaning a macro conditional would not be seeing
values like zero being written to the buffer, but references to a Matrix fields .a,
.b, .c, .d, .e, and .f.

It would be great if we could tell the compiler "Only write these values if they are
non-zero, however don't check for this during runtime".

## Detailed design

I propose a special compiler conditional statement which only exists during compile-time,
and which is evaluated late in the compilation process. I don't know what would be good syntax
for something which only exists for the compiler, but I imagine something like this:

```haxe
    hint( conditional ) {
        // outcome 1
    } else {
        // outcome 2
    }
```

Examples where this can be useful include:

* Avoiding zero writes to zero-initialized memory
* setPixel could avoid alpha-blending colors if it knows the new pixel has a completely opaque alpha channel
* Bit-fiddling math could use a faster idiv function if it knows it's given a positive integer
* Loop-unrolling for non-traditional loops when the number of iterations are known

## Impact on existing code

This should have zero impact on existing code.

## Drawbacks

It might be a burden to the Haxe language to introduce another type of conditional which is only useful in
edge cases.
