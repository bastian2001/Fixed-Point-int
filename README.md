# Fixed Point int

This library implements signed fixed point integer classes – fix32 (16.16) and fix64 (32.32) – that can be used to accelerate math functions on microcontrollers, or generally anywhere in a C++ application.

The code is currently only targeting an RP2350 and unless you, yes you, decide to change that, it will likely stay like that. The 4 basic maths functions (+-\*/) as well as bit shifting, `abs` etc. are not dependent on the RP2350 hardware, but stuff like trigonometric functions or sqrt currently require the RP2xxx interpolator. Porting for the RP2040 is easy, about 10 lines max (if any, there just needs to be a polyfill for the `__clz` (count leading zeroes) intrinsic), but other targets will require polyfills for the interpolator. Also doable, but the speed gains will be lower and effort for porting slightly higher.

This repo mostly exists for me, so that I have quick and easy access to the functions in my projects using PlatformIO.

> [!NOTE]
> Going forward, when I am referring to `fix32` AND `fix64`, I will just call it `fix` for simplicity. If something only applies to one of the types, I will call it `fix32` or `fix64`.

## Usage

1. Somehow install it in your project, e.g. by adding this repo to your `lib_deps` in PlatformIO or downloading and adding the ZIP to your Arduino project.
2. Declare `fix` numbers the same way you would normally declare ints or floats. The operators are overridden, so you can just use these numbers like any other types. `a = b * c` and so on, you know.
3. You can also use ints, floats etc. in your operations as long as the fix is the first (fix + int works, but int + fix does not). If you need that (for things like 1 / a, write `fix(1) / a`)

## Class reference

### Member variables

-   For `fix32`, the internal variable is an `int32_t`. Thus, positive as well as negative numbers can be stored. This variable can be accessed using the member variable `fix32::raw`. This is useful for using the interpolator and other things that require raw access to the internal state.
-   `fix64::raw` is therefore obviously an `int64_t`.

### Overridden operators and casts

-   Addition, subtraction, multiplication and division (`+-*/`) are properly overridden, i.e. available just like with normal number types. As long as the fix number is the first, you can also add(, subtract etc.) other number types, too. Most of the time, these operations are implemented using a simple cast to fix.
-   Assignment during addition, subtraction etc. (`+=`, `-=`) is implemented only for fix32, not for fix64 numbers. For fix64, just use `a = a + b`.
-   The operators `>>`, `<<` shift the internal raw number using the normal shift operators.
-   Unary operator `-` (e.g. `a = -b`) is supported and negates the number.
-   Casting to a bool (`bool b = (bool) a`, `if (a)`) will return `true` for any value other than _exactly_ zero.
-   Comparison operators `>`, `<`, `>=`, `<=`, `==`, `!=` are available. Just like with floats, rounding errors can occor (albeit a bit more predictable when they happen), thus use `==` and `!=` with caution.

### More member functions

-   `fix::sign()` returns an int of value 1 or -1. If the fix value is 0, this function returns 1.
-   `fix::abs()` returns the absolute value of this fix by multiplying it by its sign. Note that if this fix is at its lowest value (e.g. -32768 for fix32), no negation is possible and the return value will be -32768.
-   `fix::getf32()` converts this fix to a float. `fix::getf64()`, `fix::geti32()` and `fix::getu32()` give you a `double`, `int32_t` or `uint32_t`, respectively.
-   `fix64::getfix32()` and `fix32::getfix64()` convert this fix to the other fixed point type. Note that you can also just assign a fix32 to a fix64 and vice versa so these are not really needed.
-   `fix32 fix32::setRaw(int32_t raw)` and `fix64 fix64::setRaw(int64_t raw)` write a new raw value to the internal variable. The reason for these setters is to enable one-liners for return statements in the other functions. If you want to write the raw value, you can just directly write to the member variable. There is no difference.

## Math functions

I ported many math functions for quick usage. If one is missing and you have no idea how this function could be implemented using fix numbers, just convert the fix to a float or double, do your thing, and convert it back, e.g. `fix32 a = pow(b.getf32(), c.getf32());`. The following functions generally behave like their C counterparts (if such exists). Check their doxygen comments for info on precision.

-   `fix32 sinFix(const fix32 x);`
-   `fix32 cosFix(const fix32 x);`
-   `void sinCosFix(const fix32 x, fix32 &sinOut, fix32 &cosOut);`
-   `fix32 atanFix(fix32 x);`
-   `fix32 atan2Fix(const fix32 y, const fix32 x);`
-   `fix32 acosFix(fix32 x);`
-   `fix32 sqrtFix(fix32 x);`

### Important

These math functions need preparation for the interpolator. Call `initFixMath()` once at the beginning of your program. This fills the LUTs with the values. Also, call `startFixMath()` once before every math calculation batch. This writes the necessary interpolator configs to the interpolator. If you don't use the interpolator for anything else, you only have to call this function once (per core). See the following:

```c++
void begin(){
	initFixMath();
}
void loop(){
	fix32 someValue = Serial.readString().toFloat();
	someValue *= 3; // +-*/ and the other basics do not need startFixMath()
	startFixMath();
	fix32 iAmgRoot = sqrtFix(someValue);
	fix32 si = sinFix(iAmgRoot);
	// si = sin(sqrt(someValue * 3))
	Serial.println(si.getf32());

	// ...other stuff from your code that may or may not use the interpolator
}
```

## Application notes

-   Fixed points have a fixed precision (as in: step) for all calculations. Let's call it eps. For fix32, eps is 1/65536, and for fix64, eps is 2^-32 (roughly 1/4bn). This why they are called _fixed_. Floats and doubles have a precision that is relative to the magnitude of the number. Their decimal point (actually binary but whatever) slides (_floats_). This makes floats slower to compute but usually more versatile.
-   There are min and max values for fixed point ints. You can find them at `FIX32_MIN` (-32768), `FIX32_MAX` (+32768 - eps), `FIX64_MIN` (-2^31, roughly -2bn) and `FIX64_MAX` (+2^31 - eps, roughly +2bn).
-   There are also constants available for Pi etc: `FIX_PI`, `FIX_PI_2`, `FIX_PI_4`, `FIX_3PI_4`, `FIX_2_PI`, `FIX_1_PI`, `FIX_TWOPI`, `FIX_SQRTPI`, `FIX_SQRT2`, `FIX_SQRT1_2`, `FIX_E`, `FIX_LOG2E`, `FIX_LOG10E`, `FIX_LN2`, `FIX_LN10`
-   When assigning values, they are always rounded towards 0 to the nearest available fix step.
-   The RP2350 has an FPU. During my testing, I found that this library is still faster, but loses quite a bit of its advantage compared to back when I used it on an RP2040. The biggest advantages come in the trigonometric functions and sqrt and when using the raw values with other interpolation methods.
-   All of the basics are coded in a way that the compiler inlines them. This has two advantages:
    1.  Faster: The CPU does not need to jump into and out of a function
    2.  Constants and constexpr: The compiler automatically does as much as possible during compilation so that stuff like `fix32 a = b * 2.3` does not actually do _any_ floating point math on the microcontroller (given b is fix, too). The literal 2.3 is automatically converted to a fix at compile time!
-   Overflow: Because it is all implemented using ints, overflow happens the same way as with normal ints. If you write `fix32 a = 32769`, it will actually assign a value of -32767. Similarly with additions etc.
-   fix64 cannot be divided by another fix64. Please convert the values to fix32, floats, doubles, whatever you want. Implementing this would require dividing a 96 bit value by a 64 bit value. No clue how to do that, and, honestly speaking, this is a very rare case.

## Examples

There are none, haha. But you can check out my [flight controller](https://github.com/bastian2001/Kolibri-FC/blob/5b1625858ee2d05af3bae6ff0e256acd3157242d/Firmware/src/pid.cpp) where I use them extensively.
