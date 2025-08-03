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
