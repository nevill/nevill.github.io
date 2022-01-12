---
title: "String in Go"
date: 2022-01-11T17:08:18+08:00
draft: false
---

运行一个取字符串地址的程序
```go
// hello.go
package main

import (
	"fmt"
)

func main() {
	str := "Hello Golang!"
	strPtr := &str
	fmt.Printf("address of str = %p\n", strPtr)
	byteArray := []byte(str)
	fmt.Printf("address of byte array = %p\n", byteArray)
	fmt.Scanln()
}
```
观察到输出
```
address of str = 0xc00008a210
address of byte array = 0xc0000ac000
```
这两个地址的值并不相同。

正好在学习使用 crash utility，可以用来观察一下内存结构
```
crash> rd -32 0xc00008a210 8
      c00008a210:  004a3cfc 00000000 0000000d 00000000   .<J.............
      c00008a220:  005383e0 00000000 00538420 00000000   ..S..... .S.....

crash> rd -a 0xc0000ac000
      c0000ac000:  Hello Golang!
```
猜测，0xc00008c210 这个指针指向了一个字符串真正的地址，即 0x004a3cfc 才是存放的内存地址。
尝试读取一下：
```
crash> rd -8 0x004a3cfc 32
          4a3cfc:  48 65 6c 6c 6f 20 47 6f 6c 61 6e 67 21 4d 61 73   Hello Golang!Mas
          4a3d0c:  61 72 61 6d 5f 47 6f 6e 64 69 4d 65 6e 64 65 5f   aram_GondiMende_
```
果然找到了。

翻一下代码，有一个 stringStruct
```go
// runtime/string.go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
同时，观察到 0xc00008a218 这个地址的值 0x0000000d 即 `Hello Golang!` 字符串的长度 13。即验证了 0xc00008a210 这个地址对应的结构体。
那么通过 `[]byte(str)` 强制转换以后为什么会返回另外一个内存地址 0xc0000ac000 而没有返回 0x004a3cfc 呢？
原因也很简单， string 类型应该是不可修改的 (immutable)[^1]，也无法通过 `&str[i]` 来获取其某一个 byte 的地址。

可以查看一下汇编：
```
// go tool compile -S hello.go
// ...
	0x0085 00133 (hello.go:11)	MOVQ	"".&str+56(SP), CX
	0x008a 00138 (hello.go:11)	MOVQ	(CX), BX
	0x008d 00141 (hello.go:11)	MOVQ	8(CX), CX
	0x0091 00145 (hello.go:11)	XORL	AX, AX
	0x0093 00147 (hello.go:11)	PCDATA	$1, $0
	0x0093 00147 (hello.go:11)	CALL	runtime.stringtoslicebyte(SB)
	0x0098 00152 (hello.go:12)	CALL	runtime.convTslice(SB)
	0x009d 00157 (hello.go:12)	MOVUPS	X15, ""..autotmp_29+64(SP)
	0x00a3 00163 (hello.go:12)	LEAQ	type.[]uint8(SB), CX
	0x00aa 00170 (hello.go:12)	MOVQ	CX, ""..autotmp_29+64(SP)
	0x00af 00175 (hello.go:12)	MOVQ	AX, ""..autotmp_29+72(SP)
// ...
```
这里调用了一个 runtime.stringtoslicebyte
```go
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}
```
可以看到，最后会调用 `copy` 将源内容复制到一个新的 `[]byte` 切片中。

总结一下：
1) string 类型结构仅含两个元素，一个指向具体内容的指针和字符串长度
2) string 类型是不可修改的，所以在强制转换类型的时候发生了复制
3) 写这篇主要目的是展示 crash utility 怎么用 :)

参考：
[^1] https://go.dev/ref/spec#String_types
