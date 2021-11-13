---
layout: post
title: Golang - Precise error testing and internationalization (i18n)
comments: true
---

# Golang - Precise error testing and internationalization (i18n)

Go has great built-in support for testing code. One would simply run in the root of the project `go test` and all the tests of every package within the current module will be invoked.

Let's look at the following code:
```go
func Divide(l int, r int) (int, error) {
	if r == 0 {
		return 0, fmt.Errorf("Cannot divide %d by zero", l)
	}
	return l / r, nil
}
```

The `Divide` function receives two arguments; left and right operands. If the right operand is zero, then it returns `0` for the result and error containing a useful message, otherwise it returns the result and no error.

Looking at this code already gives an idea of how to test it:
- Proper division - check that the returned value is as expected and no error is returned.
- Divide by zero - pass `0` as the right operand and check that the returned value is `0` and the error message is as expected.


The test code would look similar to this:
```go
func TestDivide(t *testing.T) {
	testcases := []struct {
		name         string
		leftOperand  int
		rightOperand int
		expectedRet  int
		expectedErr  error
	}{
		{
			name:         "Good: proper division",
			leftOperand:  8,
			rightOperand: 4,
			expectedRet:  2,
			expectedErr:  nil,
		},
		{
			name:         "Bad: divide by zero",
			leftOperand:  8,
			rightOperand: 0,
			expectedRet:  0,
			expectedErr:  fmt.Errorf("Cannot divide %d by zero", 8),
		},
	}

	for _, tc := range testcases {
		t.Run(tc.name, func(t *testing.T) {
			actualRet, actualErr := Divide(tc.leftOperand, tc.rightOperand)
			if actualErr != nil {
				require.NoError(t, tc.expectedErr)
				require.Contains(t,
					strings.ToLower(tc.expectedErr.Error()),
					strings.ToLower(actualErr.Error()))
				require.Equal(t, tc.expectedRet, 0)
			} else {
				require.Error(t, tc.expectedErr)
				require.Equal(t, tc.expectedRet, actualRet)
			}
		})
	}
}
```

Although this works, there are multiple caveats with this approach:
1. If the returned error message in the `Divide` function is altered in future updates, this means the test cases require updating as well, otherwise the tests will fail.
2. If the tested function returns as a complex formatted error message, with multiple arguments appended to it; it will usually require checking if the error message contains a part of the error message either using `require.Contains` or `require.True` with `strings.HasPrefix|Suffix`.
3. When using `require.Contains`, it could possibly cause false positives where the expected error message contains part of the tested / actual error message. This can happen if the actual error message was propagated from another sub-function (this is not the case for the `Divide` function, although in complex scenarios this could happen).

It is likely there may be more caveats to the approach described above, or perhaps even a different way of testing this all together, however for the purpose of this blog post I'll mainly focus on the points described above.

The problem with [1], [2] and [3] would be better approached if Go had some built-in mechanism to define errors with error-codes. An interface such as `fmt.Errorf(1001, "My error")` wouldn't result in duplicating the hardcoded error-messages in the test cases. Consequently, error-codes will be tested instead of duplicating and relying on the hardcoded error-messages.

Checking for error-codes still isn't ideal, especially if it's a very complex formatted error message (as mentioned in problem [2] and [3]). What if there was an easier and more precise way to reconstruct the error with its error-code and format message string?

I wrote a simple Proof Of Concept (PoC) for the purpose of demonstrating how to solve these problems and at the same time benefit from allowing convenient ways to internationalize all error-messages with few simple changes. The PoC code can be found in the following url: [https://github.com/giladreich/go-errori18n](https://github.com/giladreich/go-errori18n)

The way the PoC approaches this problem is by defining a custom error structure (`Error`) using similar constructs to the built-in `error`:
```go
type Error struct {
	ErrorCode ErrorCode
	args      []interface{}
}

func (e *Error) String() string {
	return fmt.Sprintf(e.ErrorCode.String(), e.args...)
}

func Errorf(code ErrorCode, args ...interface{}) *Error {
	return &Error{
		ErrorCode: code,
		args:      args,
	}
}
```

The `Divide` function will now return a `*Error` instead of `error`:
```go
import (
	. "github.com/giladreich/go-errori18n/errori18n"
)

func Divide(l int, r int) (int, *Error) {
	if r == 0 {
		return 0, Errorf(ERR_DIVIDE_BY_ZERO, l)
	}
	return l / r, nil
}
```

Note that the `.` notation is used for importing the `go-errori18n` package, which will essentially allow for the predefined error-codes to be used without having to specify its full package definition (`go-errori18n.ERR_DIVIDE_BY_ZERO`). Additionally, using the `Errorf`, similar to `fmt.Errorf`, returns as a `*Error` instead of `error`.

Now that the predefined error-code (`ERR_DIVIDE_BY_ZERO`) is used, it allows the code to both scale for internationalization and benefit from writing simple and precise test cases:
```go
func TestDivide(t *testing.T) {
	testcases := []struct {
		name         string
		leftOperand  int
		rightOperand int
		expectedRet  int
		expectedErr  *Error
	}{
		{
			name:         "Good: proper division",
			leftOperand:  8,
			rightOperand: 4,
			expectedRet:  2,
			expectedErr:  nil,
		},
		{
			name:         "Bad: divide by zero",
			leftOperand:  8,
			rightOperand: 0,
			expectedRet:  0,
			expectedErr:  Errorf(ERR_DIVIDE_BY_ZERO, 8),
		},
	}

	// Need to change language during runtime? no problem.
	SetLanguage(LANG_DE)

	for _, tc := range testcases {
		t.Run(tc.name, func(t *testing.T) {
			actualRet, actualErr := Divide(tc.leftOperand, tc.rightOperand)
			if actualErr != nil {
				require.NotNil(t, tc.expectedErr)

				// Either test against error codes
				require.Equal(t, tc.expectedErr.ErrorCode, actualErr.ErrorCode)

				// or strings for nice output in failed tests
				require.Equal(t, tc.expectedErr.String(), actualErr.String())

				require.Equal(t, tc.expectedRet, 0)
			} else {
				require.Nil(t, tc.expectedErr)
				require.Equal(t, tc.expectedRet, actualRet)
			}
		})
	}
}
```

When needing to add more error-codes (`ErrorCode`), the `errori18n/language/en.yml` should be edited, e.g.:
```yaml
ERR_DIVIDE_BY_ZERO: "Cannot divide %d by zero"
ERR_ANOTHER_MSG: "This is my new error message"
```

and then using the `errori18n/codesgen.go` tool to re-generate the error-codes:
```sh
go run errori18n/codesgen.go
```

Depending on which automation a given project has, the invocation of the `codesgen.go` tool can be automated using `//go:generate` directives and then calling the `go generate` command.
