---
title: golang函数式编程一点尝试
date: 2018-03-13 14:18:26
tags: [golang, 函数式]
---

java8 引入lambda表达式和Stream相关的api来实现(一定程度的)函数式编程，golang func作为first class function，可以作为函数的参数和返回值，也可以赋给变量，因此实现函数式编程也不困难。


但是要吐槽一下，由于没有泛型，所以抽象只能使用interface{}加反射来做，完全失去了golang作为静态语言的优势。

<!--more-->

#### 功能代码

```golang
package main

import (
	"errors"
	"reflect"
)

type FilterFn func(interface{}) bool
type MapFn func(interface{}) interface{}
type ReduceFn func(interface{}, interface{}) interface{}
type CompareFn func(interface{}, interface{}) int

func Filter(fn FilterFn, input interface{}, output interface{}) error {
	if fn == nil || input == nil || output == nil {
		return errors.New("输入的参数不对")
	}
	inputValue := reflect.ValueOf(input)
	outputValue := reflect.ValueOf(output).Elem()
	inputType := reflect.TypeOf(input).Kind()
	outputType := reflect.TypeOf(output).Kind()

	if inputType != reflect.Slice || outputType != reflect.Ptr {
		return errors.New("input必须是slice， output必须是slice的指针")
	}

	for i := 0; i < inputValue.Len(); i++ {
		one := inputValue.Index(i).Interface()
		if fn(one) {
			outputValue.Set(reflect.Append(outputValue, reflect.ValueOf(one)))
		}
	}
	return nil
}

func Map(fn MapFn, input interface{}, output interface{}) error {
	if fn == nil || input == nil || output == nil {
		return errors.New("输入的参数不对")
	}
	inputValue := reflect.ValueOf(input)
	outputValue := reflect.ValueOf(output).Elem()
	inputType := reflect.TypeOf(input).Kind()
	outputType := reflect.TypeOf(output).Kind()

	if inputType != reflect.Slice || outputType != reflect.Ptr {
		return errors.New("input必须是slice， output必须是slice的指针")
	}

	for i := 0; i < inputValue.Len(); i++ {
		one := inputValue.Index(i).Interface()
		outputValue.Set(reflect.Append(outputValue, reflect.ValueOf(fn(one))))
	}
	return nil
}

func Reduce(fn ReduceFn, input interface{}, output interface{}) error {
	if fn == nil || input == nil || output == nil {
		return errors.New("输入的参数不对")
	}
	inputValue := reflect.ValueOf(input)
	outputValue := reflect.ValueOf(output).Elem()
	inputType := reflect.TypeOf(input).Kind()
	outputType := reflect.TypeOf(output).Kind()

	if inputType != reflect.Slice || outputType != reflect.Ptr {
		return errors.New("input必须是slice， output必须是指针")
	}

	if inputValue.Len() == 0 {
		return errors.New("input长度必须大于等于0")
	}

	var result interface{} = nil
	for i := 0; i < inputValue.Len(); i++ {
		one := inputValue.Index(i).Interface()
		if result == nil {
			result = one
		} else {
			result = fn(result, one)
		}
	}
	outputValue.Set(reflect.ValueOf(result))
	return nil
}

func Max(fn CompareFn, input interface{}, max interface{}) error {
	return Reduce(func(o1 interface{}, o2 interface{}) interface{} {
		if fn != nil && fn(o1, o2) > 0 {
			return o1
		} else {
			return o2
		}
	}, input, max)
}

func Min(fn CompareFn, input interface{}, min interface{}) error {
	return Reduce(func(o1 interface{}, o2 interface{}) interface{} {
		if fn != nil && fn(o1, o2) < 0 {
			return o1
		} else {
			return o2
		}
	}, input, min)
}

func SumInt(input interface{}) int {
	var sum int
	fn := func(o1 interface{}, o2 interface{}) interface{} {
		return o1.(int) + o2.(int)
	}
	err := Reduce(fn, input, &sum)
	if err != nil {
		return 0
	} else {
		return sum
	}

}

func SumFloat(input interface{}) float64 {
	var sum float64
	fn := func(o1 interface{}, o2 interface{}) interface{} {
		return o1.(float64) + o2.(float64)
	}
	err := Reduce(fn, input, &sum)
	if err != nil {
		return 0
	} else {
		return sum
	}
}

func Contains(array interface{}, target interface{}) bool {
	arrayValue := reflect.ValueOf(array)
	switch reflect.TypeOf(array).Kind() {
	case reflect.Slice, reflect.Array:
		for i := 0; i < arrayValue.Len(); i++ {
			if arrayValue.Index(i).Interface() == target {
				return true
			}
		}
	case reflect.Map:
		if arrayValue.MapIndex(reflect.ValueOf(target)).IsValid() {
			return true
		}
	}
	return false
}
```


#### 测试代码

```golang
package main

import (
	"strconv"
	"testing"
)

func TestFilter(t *testing.T) {
	input := []int{1, 2, 3, 4}
	var output []int
	err := Filter(func(x interface{}) bool {
		return x.(int)%2 == 0
	}, input, &output)
	if err == nil && output[0] == 2 && output[1] == 4 {

	} else {
		t.Error("错误")
	}
}

func TestMap(t *testing.T) {
	input := []int{1, 2, 3, 4}
	var output []string
	err := Map(func(x interface{}) interface{} {
		return strconv.FormatInt(int64(x.(int)), 10)
	}, input, &output)
	if err == nil && output[0] == "1" && output[1] == "2" {

	} else {
		t.Error("错误")
	}
}

func TestSumInt(t *testing.T) {
	input := []int{1, 2, 3, 4}
	result := SumInt(input)
	if result == 10 {

	} else {
		t.Error("错误")
	}
}

func TestSumFloat(t *testing.T) {
	input := []float64{1, 2, 3, 4}
	result := SumFloat(input)
	if result == 10 {

	} else {
		t.Error("错误")
	}

	input = []float64{}
	result = SumFloat(input)
	if result == 0 {

	} else {
		t.Error("错误")
	}
}

func TestMax(t *testing.T) {
	input := []int{1, 2, 3, 4}
	max := 0
	err := Max(func(x interface{}, y interface{}) int {
		a1 := x.(int)
		a2 := y.(int)
		if a1-a2 < 0 {
			return -1
		} else if a1 == a2 {
			return 0
		} else {
			return 1
		}

	}, input, &max)
	if err == nil && max == 4 {

	} else {
		t.Error("错误")
	}
	input2 := []string{"a", "b", "cc"}
	var max2 string
	err = Max(func(x interface{}, y interface{}) int {
		a1 := len(x.(string))
		a2 := len(y.(string))
		if a1-a2 < 0 {
			return -1
		} else if a1 == a2 {
			return 0
		} else {
			return 1
		}

	}, input2, &max2)
	if err == nil && max2 == "cc" {

	} else {
		t.Error("错误")
	}
}

func TestMin(t *testing.T) {
	input := []int{1, 2, 3, 4}
	min := 0
	err := Min(func(x interface{}, y interface{}) int {
		a1 := x.(int)
		a2 := y.(int)
		if a1-a2 < 0 {
			return -1
		} else if a1 == a2 {
			return 0
		} else {
			return 1
		}

	}, input, &min)
	if err == nil && min == 1 {

	} else {
		t.Error("错误")
	}
	input2 := []string{"a", "bb", "cc"}
	var min2 string
	err = Min(func(x interface{}, y interface{}) int {
		a1 := len(x.(string))
		a2 := len(y.(string))
		if a1-a2 < 0 {
			return -1
		} else if a1 == a2 {
			return 0
		} else {
			return 1
		}

	}, input2, &min2)
	if err == nil && min2 == "a" {

	} else {
		t.Error("错误")
	}
}

func TestReduce(t *testing.T) {
	input := []string{"1", "2", "3"}
	var result string
	err := Reduce(func(x interface{}, y interface{}) interface{} {
		return x.(string) + y.(string)
	}, input, &result)
	if err == nil && result == "123" {

	} else {
		t.Error("错误")
	}
}

func TestContains(t *testing.T) {
	input := []string{"1", "2", "3"}
	target := "1"
	if Contains(input, target) {

	} else {
		t.Error("错误")
	}

	target = "4"
	if Contains(input, target) {
		t.Error("错误")
	}
}
```