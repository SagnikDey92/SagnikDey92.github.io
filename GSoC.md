
<!-- ---
title: gsoc19_Final_Eval
--- -->
# Google Summer of Code 2019: Boost.Real
## Introduction
I was involved in getting the library Boost.Real to review ready state. It is a C++ library that tries to get rid of untracked errors caused by traditional floating point arithmetic by using range arithmetic. The library was started in GSoC'18 by Laouen Belloli,  and me along with Kimberly Swanson worked on it this year.

## Commit Summary with Links (trivial commits omitted)
### Merged commits
[Minor typo corrections. Commited before the start of GSoC](https://github.com/BoostGSoC19/Real/commit/225e08025709eaef8da81a5118d1148b0fcbb312)

[Made some changes to enforce user declaration of max_precision variable](https://github.com/BoostGSoC19/Real/commit/095e3e97f97d9f0f6b6df3f863e638e461a03fb6)

[Adding User Defined Literals. Changed  string parser code into regex](https://github.com/BoostGSoC19/Real/commit/68a0f031629e0e9355f5b41144f668fcb4195a37)

[Added tests for the User Defined Literals](https://github.com/BoostGSoC19/Real/commit/51f9f3185ef21420bdda9efac2e96aa29dadf073)

[Increased the use of regex in code by using regex replace for splitting strings](https://github.com/BoostGSoC19/Real/commit/04d8adcdac089eb3908f8cf4c6a182ef285bfa82)

[Adding base change](https://github.com/BoostGSoC19/Real/commit/0a7bd000833a84d846d7deeef55465c4b2e9e0d1)

[All tests modified](https://github.com/BoostGSoC19/Real/commit/0ce78b30977c66e83d48aaecb4d572dee4aaeb7a)

[Base change finally done at this point](https://github.com/BoostGSoC19/Real/commit/8b96c9dcf07117e987cb97eb49ad9cb854fd62fe)

[Updating catch2 version and adding template parameterization to test cases](https://github.com/BoostGSoC19/Real/commit/d442d5211cb061c80bd1251d38a1fa76d0220459)

**The complete list of merged commits can be found at this link**:
[https://github.com/BoostGSoC19/Real/commits?author=SagnikDey92](https://github.com/BoostGSoC19/Real/commits?author=SagnikDey92)
### Unmerged closed PRs
[https://github.com/BoostGSoC19/Real/pull/6](https://github.com/BoostGSoC19/Real/pull/6)

[https://github.com/BoostGSoC19/Real/pull/18](https://github.com/BoostGSoC19/Real/pull/18)

[https://github.com/BoostGSoC19/Real/pull/19](https://github.com/BoostGSoC19/Real/pull/19)

[https://github.com/BoostGSoC19/Real/pull/23](https://github.com/BoostGSoC19/Real/pull/23)

[https://github.com/BoostGSoC19/Real/pull/24](https://github.com/BoostGSoC19/Real/pull/24)

[https://github.com/BoostGSoC19/Real/pull/26](https://github.com/BoostGSoC19/Real/pull/26)
### PRs remaining to be merged
[Final bug fix for the division operator.](https://github.com/BoostGSoC19/Real/pull/38)

## My Tasks
This varied a bit from my proposal since I found out I was to be collaborating with another GSoC student, Kimberly Swanson. We communicated through group emails including us and our mentors, Damian Vicino and Laouen Belloli. We split up the tasks. Here are the ones that I was assigned:

* Implement base change for internal representation.
* Implement User Defined Literal method for declaring objects of type real.
* Redesign test cases.
* Add Karatsuba multiplication (ongoing)

The base change task ended up involving a lot more changes than I had originally anticipated and thus took up most of my time. This is primarily because it required a major refactor to add templating. Also, while Kimberly had made the division algorithm, she had to rush it since it was a dependency for the base change task. As such, it was initially quite buggy and I contributed towards several bug fixes in it.

As for the Karatsuba multiplication, I got it working when in base 10 and sent a PR but I haven't been able to get it to work in the new base yet and thus it's not present in the code currently.

## Base Change
This task involved changing the number base used internally from decimal (base 10) to base ```INT_MAX```. This is because, originally, the library was using a ```vector<int>``` to store the real numbers but the base was 10. So, the capacity of integers was being wasted since each digit is from ```0 - 10``` but the integer variable can store upto ```INT_MAX-1```. This problem can been eliminated by changing the number base used internally from 10 to ```INT_MAX```. However, it was decided to change the ```vector<int>``` to ```vector<T>``` where ```T``` is a template parameter. Thus the internal base actually needs to be the max capacity of the parameter ```T``` for optimal space utilisation.

## Additional tasks due to base change

I had originally thought that the only things to be done in base_change were to write some helper functions to change the base to and from ```INT_MAX```. The base had to be changed since, in base 10, something like 123 stored as a vector takes three integers storing 1, 2 and 3. This wastes a lot of memory. Thus, the base has been changed to ```INT_MAX``` to utilise space optimally.

First of all, the mentors suggested adding templating to the entire library to use a custom data type for storing the digits and not just int. This required me to add templating to the entire library and also this kept causing issues when trying to merge since, at the time, Kimberly too was working on a major refactor.

Next, there was an issue of overflowing in intermediate calculations with the new base which had not crossed my mind while making the proposal due to which all of the mathematical operations had to be changed. Of course, I'm not talking about just the vector operations needing to be modified to work in the new base. For example: 

```
int digit = carry + lhs_digit + rhs_digit;
if (digit > 9) {
	carry = 1;
	digit -= 10;
} else {
	carry = 0;
}
```
This is a line from the previous implementation of the library. This works fine in case the digits are between 0 to 9 as in base 10. But in base ```INT_MAX```, the digit variable might overflow. The problem is even more pronounced in the previous multiplication:

```
int sum = lhs[i]*rhs[j] + result[i_n1 - i_n2] + carry;
// Carry for next iteration
carry = sum / 10;
// Store result
result[i_n1 - i_n2] = sum % 10;
i_n2++;
```

The variable sum can overflow by quite a lot. However the carry, which in the new implementation would be ```carry = sum / INT_MAX```, is guaranteed to never overflow. Similarly, ```sum%INT_MAX``` will never overflow. So to find these while avoiding overflows, I had to write two helper functions:

```
//Returns (a*b)%mod
T mulmod(T a, T b, T mod) {
	T res = 0; // Initialize result
	a = a % mod;
	while (b > 0) {
		// If b is odd, add 'a' to result
		if (b % 2 == 1)
		res = (res + a) % mod;
		// Multiply 'a' with 2
		a = (a * 2) % mod;
		// Divide b by 2
		b /= 2;
	}
	return res % mod;
}
```
```
//Returns (a*b)/mod
T mult_div(T a, T b, T c) {
	T rem = 0;
	T res = (a / c) * b;
	a = a % c;
	// invariant: a_orig * b_orig = (res * c + rem) + a * b
	// a < c, rem < c.
	while (b != 0) {
		if (b & 1) {
			rem += a;
			if (rem >= c) {
				rem -= c;
				res++;
			}
		}
		b /= 2;
		a *= 2;
		if (a >= c) {
			a -= c;
			res += b;
		}
	}
	return res;
}
```

Several other changes also needed to be made to functions here and there to remove overflow completely from the library.

**NOTE**: *I haven't able to make the base ```INT_MAX```  and was only able to take it up to ```INT_MAX/2```. This is because I'm still getting integer overflows during the multiplication operation. This needs to be fixed as one bit of memory is wasted per integer digit as the digits in base ```INT_MAX/2``` can only go up to ```INT_MAX/2 - 1```.* 

### Redesigning the test cases
Another major task that I had to do along with the base change that I wasn't expecting nor looking forward to, to be honest, was essentially having to rewrite all the test cases to work in the new base. For this, I had to update the catch2 version we were using as well since the mentors wanted the test cases to be type parameterized to test for templating issues in all data types. This wasn't possible in the previous version of catch we were using. Now, the base used internally is dependent on a template parameter which determines the data type used for storing digits of the Boost.Real number. And the new test cases run all the tests over three values of this template parameter, namely: ```int```, ```long``` and ```long  long```.

## Implement User Defined Literal method for declaring objects of type real.

This task basically is to add functionality to declare Boost.Real type variable using User Defined Literals. This enables us to declare reals like this:
```auto a = 123_r;	//This initialises a by a Boost.Real variable with value 123.```
This ended up being a really trivial task since all I had to do was add the lines:
```
//User Defined Literals

inline auto operator "" _r(long double x) {
	return boost::real::real<int>(std::to_string(x));
}

inline auto operator "" _r(unsigned long long x) {
	return boost::real::real<int>(std::to_string(x));
}

inline auto operator "" _r(const char* x, size_t len) {
	return boost::real::real<int>(x);
}
```

## Adding Karatsuba Multiplication
As the title suggests, this task involved me adding Karatsuba Multiplication feature to the library. This is a divide and conquer approach to vector multiplication that has a theoretical time complexity better than the naïve long vector multiplication algorithm implemented earlier. I was able to do this in the base 10 case. After base change however, I have been unsuccessful in getting it to work. Though, when testing in the base 10 case, the karatsuba does seem to take more time than the regular long multiplication, so some benchmarking needs to be done to check in which cases this is actually useful. Of course, the choice of whether to use it or not will be left to the user via setting a global variable.

## Bugs in Division
I had asked Kimberly to finish the Division code a bit ahead of schedule since it was needed for the base change. The reason is that after base change, something like 1.23 is stored internally as an operation number as ```123/100```. The reason for that is that finitely representable numbers in base 10 may not be finitely representable in other bases.

This led to a bit rushed development of the division code and led to several bugs creeping up here and there. Due to the crucial role of division in base change, I could not leave these bugs unsolved and often had to collaborate with Kimberly to get the bugs fixed. 

Even now I feel there is much room for improvement in the division algorithm and I fixed a major bug quite recently. The fix itself was just one line but took me a really long time to find. With that fixed, the library should finally be ready for review.
## Adding regex
I thought using regular expressions library for pattern matching would clean up some of the code so I set about this task which was not in my proposal.
This, however,  ended up being a futile effort. I had put in some time trying to use regex to clean up the code for string parsing. This also got rid of several loopholes in illegal string exception throwing. However, it was finally decided that regex is too unpredictable, running differently depending on the environment and even the Travis build was failing due to the regex for MacOS. Thus this idea was finally scrapped, and we have reverted to doing the string parsing manually without using regex.

## Areas for Future improvement
### Base conversions
Since I wanted just to get the base change done to see if it works, as a base implementation, I have implemented quite basic code to do the base conversions. These end up taking up a lot of computation time. I think these base conversions can be optimised.

### More operations
Operations like exponent, square root etc. should be added to the library at some point. 

### Perfect the base change
As I mentioned above, the base being used internally currently is ```INT_MAX/2``` as I wasn't able to get it working for base ```INT_MAX```. I'll have to look into it and perfect it later.


# Final Remarks
I thoroughly enjoyed this summer, courtesy of GSoC, and it has also been a great learning experience for having been able to interact with highly experienced and talented mentors like Laouen and Damian. I was afraid that since the base change task was getting out of hand, I might not be able to complete everything from my proposal initially. However, fortunately, I was working alongside another talented GSoC student Kimberly Swanson who helped a lot in getting the project to completion by sharing tasks mutually. I plan to stick around until the code is finally deployed and am hoping for even more enriching interactions with the Boost community.
