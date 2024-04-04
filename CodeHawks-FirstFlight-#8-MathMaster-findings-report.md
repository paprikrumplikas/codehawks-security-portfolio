# First Flight #8: Math Master - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect overflow protection logic in `MathMasters::mulWadUp` results in false positives](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect overflow protection logic in `MathMasters::mulWadUp` results in false positives            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L52

## Summary

Incorrect implementation of overflow protection in `MathMasters::mulWadUp` dilutes the intended safety check against overflow, so that certain non-overflow multiplication operations get treated as if they were overflows (false positives).

## Vulnerability Details
The following line of code
```
            // Equivalent to `require(y == 0 || x <= type(uint64).max / y)`.
            if mul(y, gt(x, or(div(not(0), y), x))) {
```
is supposed to be a safety check designed to prevent multiplication overflow when performing fixed-point arithmetic. Although the comment above it describes a correct overflow check, the assembly code implementation is incorrect.
The `or` operation performs a bitwise `OR` between `div(not(0), y)` and `x`, combining the bits of `x` and the safe maximum multiplier for `y`. This does not make logical sense in the context of assessing the risk of multiplication overflow: since `or(div(not(0), y), x)) >= x`, the inclusion of the `OR` operation effectively dilutes the intended safety check against overflow, resulting in false positives.

Example: using uint64 instead of uint256 for simplicity, `mulWadUp` incorrectly throws an overflow error (in uint64 context) for `x = 3008292032 (0xb34ee4c0)` with `y = 2991730824 (0xb2523088)`.

## Impact
The flawed logic may erroneously treat certain non-overflow multiplication operations as if they were overflows, leading to unnecessary transaction reverts. 
Such false positives in overflow detection not only compromise the contract's operational integrity but also could lead to a loss of user trust and potential financial disruptions for applications depending on the contract's accurate execution of arithmetic operations.

## Tools Used
Halmos, ChatGPT

## Recommendations
Correct the overflow safety check, e.g.

```diff
-       if mul(y, gt(x, or(div(not(0), y), x))) {
+      if mul(y, gt(x, div(not(0), y))) {
```

(This correct version is being used in `mulWad˛.)
		





