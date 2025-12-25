What happened?
When parsing very large quantity values, i.e., values close to math.MaxInt64, k8s.io/apimachinery/pkg/api/resource.MustParse is unable to correctly interpret them.
 Similar parsing failures also occur when these high values are used within or passed to CEL validation rules.
 
 MY APPROACH:

test run:
```go
q := resource.MustParse(strconv.FormatInt(math.MaxInt64, 10))
fmt.Println(q.AsInt64())

fmt.Println(resource.NewQuantity(math.MaxInt64, resource.DecimalSI).AsInt64())
```

output:
```
0 false// this is incorrect as it should return true
9223372036854775807 true
```

SOLUTION TILL NOW
no logs so problem comes directly from library as mentioned

```go
func MustParse(str string) Quantity {
        q, err := ParseQuantity(str)
        if err != nil {
                panic(fmt.Errorf("cannot parse '%v': %v", str, err))
        }
        return q
}
```

-function does not panic

examine ParseQuantity(str)// ParseQuantity turns str into a Quantity, or returns an error.

```go
func parseQuantityString(str string) (positive bool, value, num, denom, suffix string, err error) {
	positive = true
	pos := 0
	end := len(str)

	// handle leading sign
	if pos < end {
		switch str[0] {
		case '-':
			positive = false
			pos++
		case '+':
			pos++
		}
	}

	// strip leading zeros
Zeroes:
	for i := pos; ; i++ {
		if i >= end {
			num = "0"
			value = num
			return
		}
		switch str[i] {
		case '0':
			pos++
		default:
			break Zeroes
		}
	}

	// extract the numerator
Num:
	for i := pos; ; i++ {
		if i >= end {
			num = str[pos:end]
			value = str[0:end]
			return
		}
		switch str[i] {
		case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
		default:
			num = str[pos:i]
			pos = i
			break Num
		}
	}

	// if we stripped all numerator positions, always return 0
	if len(num) == 0 {
		num = "0"
	}

	// handle a denominator
	if pos < end && str[pos] == '.' {
		pos++
	Denom:
		for i := pos; ; i++ {
			if i >= end {
				denom = str[pos:end]
				value = str[0:end]
				return
			}
			switch str[i] {
			case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
			default:
				denom = str[pos:i]
				pos = i
				break Denom
			}
		}
		// TODO: we currently allow 1.G, but we may not want to in the future.
		// if len(denom) == 0 {
		// 	err = ErrFormatWrong
		// 	return
		// }
	}
	value = str[0:pos]

	// grab the elements of the suffix
	suffixStart := pos
	for i := pos; ; i++ {
		if i >= end {
			suffix = str[suffixStart:end]
			return
		}
		if !strings.ContainsAny(str[i:i+1], "eEinumkKMGTP") {
			pos = i
			break
		}
	}
	if pos < end {
		switch str[pos] {
		case '-', '+':
			pos++
		}
	}
Suffix:
	for i := pos; ; i++ {
		if i >= end {
			suffix = str[suffixStart:end]
			return
		}
		switch str[i] {
		case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
		default:
			break Suffix
		}
	}
	// we encountered a non decimal in the Suffix loop, but the last character
	// was not a valid exponent
	err = ErrFormatWrong
	return
}

// ParseQuantity turns str into a Quantity, or returns an error.
func ParseQuantity(str string) (Quantity, error) {
	if len(str) == 0 {
		return Quantity{}, ErrFormatWrong
	}
	if str == "0" {
		return Quantity{Format: DecimalSI, s: str}, nil
	}

	positive, value, num, denom, suf, err := parseQuantityString(str)
	if err != nil {
		return Quantity{}, err
	}

	base, exponent, format, ok := quantitySuffixer.interpret(suffix(suf))
	if !ok {
		return Quantity{}, ErrSuffix
	}

	precision := int32(0)
	scale := int32(0)
	mantissa := int64(1)
	switch format {
	case DecimalExponent, DecimalSI:
		scale = exponent
		precision = maxInt64Factors - int32(len(num)+len(denom))
	case BinarySI:
		scale = 0
		switch {
		case exponent >= 0 && len(denom) == 0:
			// only handle positive binary numbers with the fast path
			mantissa = int64(int64(mantissa) << uint64(exponent))
			// 1Mi (2^20) has ~6 digits of decimal precision, so exponent*3/10 -1 is roughly the precision
			precision = 15 - int32(len(num)) - int32(float32(exponent)*3/10) - 1
		default:
			precision = -1
		}
	}
```

-after examination the actual problem seems to be `precision`

**Notes:**

This is a library limitation, not a panic or crash.

NewQuantity should be used for known int64 values as a workaround.

**Current Status:**

No patch applied yet; investigation only.

Master branch is unaffected.

CEL validation fails for these high values due to the same limitation.

//i am running all tests directly here since this is not master branch and i want to work directly with the codebase
