## Go最大能创建多大的数组？

前几天，在写json反序列化时，服务端返回了一个非常大的数据，大约有七八万条。

当时在想，java的数据长度是固定的，最大是2^31-1，那么golang的呢？

这几天忙完了业务处理，准备测试下，看下最长的长度。

先说结论：

*go能够创建的数组长度大小，取决于类型和平台，在不同平台上，值是不同的*



## 测试

首先创建一个数组

```golang
arr:=make([]bool,math.MaxInt32)
fmt.Println(len(arr))
```

运行并获取结果

```
2147483647
```

根据结果判断，长度达到2^31-1没问题。

继续改动

```golang
arr:=make([]bool,math.MaxUint32)
fmt.Println(len(arr))
```

发现依然正确的输出了结果

```
4294967295
```

这个值是2^32-1，已经超过了java的最大长度！

还能不能再大点？

```golang
arr:=make([]bool,math.MaxInt64)
fmt.Println(len(arr))
```

这次报错了，提示

```
panic: runtime error: makeslice: len out of range
```

但是！如果我把bool类型，替换成struct{}类型呢？

```golang
arr:=make([]struct{},math.MaxInt64)
fmt.Println(len(arr))
```

成功创建

```
9223372036854775807
```

再大点呢？

```
arr:=make([]struct{},math.MaxUint64)
fmt.Println(len(arr))
```

panic都没有，直接在编译期报错

```
./main.go:9:11: len argument too large in make([]struct {})
```

看来长度还是有点玄学。

## 源码

通过参考[1]可知，创建slice会在编译器进行处理，大致代码为

```golang
# https://github.com/golang/go/blob/go1.15.6/src/cmd/compile/internal/gc/universe.go#L57
var builtinFuncs = [...]struct {
    name string
    op   Op
}{
    ......
  // 对应make的是OMAKE操作
    {"make", OMAKE},
   ......
}
# https://github.com/golang/go/blob/go1.15.6/src/cmd/compile/internal/gc/typecheck.go#L1713
case OMAKE:
		......
		switch t.Etype {
		......
		case TSLICE:
      ......
      // 参数校验
      if !checkmake(t, "len", &l) || r != nil && !checkmake(t, "cap", &r) {
				n.Type = nil
				return n
			}
      ......
      // 将具体操作方法定义为OMAKESLICE
      n.Op = OMAKESLICE
# https://github.com/golang/go/blob/go1.15.6/src/cmd/compile/internal/gc/typecheck.go#L3710
func checkmake(t *types.Type, arg string, np **Node) bool {
	......
	switch consttype(n) {
	case CTINT, CTRUNE, CTFLT, CTCPLX:
		......
    // 如果当前值大于了maxint值，直接返回错误;
    // 跟我们创建MaxUint64的struct{} slice报错一致
		if v.Cmp(maxintval[TINT]) > 0 {
			yyerror("%s argument too large in make(%v)", arg, t)
			return false
		}
	}
```

把操作符变成`OMAKESLICE`后，我们查看具体的创建流程，发现是通过walkexpr函数进行处理，然后通过mkcal1l调用的makeslice函数执行的创建

```golang
# https://github.com/golang/go/blob/go1.15.6/src/cmd/compile/internal/gc/walk.go#L1317
case OMAKESLICE:
		......
    // n escapes; set up a call to makeslice.
    // When len and cap can fit into int, use makeslice instead of
    // makeslice64, which is faster and shorter on 32 bit platforms.

    len, cap := l, r
		// 调用函数为makeslice64
    fnname := "makeslice64"
    argtype := types.Types[TINT64]

    // 如果slice的长度和cap都是int型，使用makeslice,提升在32bit机器上的性能
    if (len.Type.IsKind(TIDEAL) || maxintval[len.Type.Etype].Cmp(maxintval[TUINT]) <= 0) &&
    (cap.Type.IsKind(TIDEAL) || maxintval[cap.Type.Etype].Cmp(maxintval[TUINT]) <= 0) {
      fnname = "makeslice"
      argtype = types.Types[TINT]
    }

    m := nod(OSLICEHEADER, nil, nil)
    m.Type = t

    fn := syslook(fnname)
    m.Left = mkcall1(fn, types.Types[TUNSAFEPTR], init, typename(t.Elem()), conv(len, argtype), conv(cap, argtype))
    ......
```

之后需要查看makeslice函数具体的判断逻辑了

```golang
# https://github.com/golang/go/blob/go1.15.6/src/runtime/slice.go#L83
func makeslice(et *_type, len, cap int) unsafe.Pointer {
  // 数组大小=数据类型大小*数组容量
  // 如果数组大小超出了go支持的MaxUintptr值，就overflow了
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// 只是为了可读性，如果数据len都超出了，报长度不支持的错误
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}

func makeslice64(et *_type, len64, cap64 int64) unsafe.Pointer {
	len := int(len64)
	if int64(len) != len64 {
		panicmakeslicelen()
	}

	cap := int(cap64)
	if int64(cap) != cap64 {
		panicmakeslicecap()
	}
	// makeslice64最底层还是会调用makeslice
	return makeslice(et, len, cap)
}
```

可以看到，数组长度是否会报错，取决于三个条件

1. 申请的数组空间大小数值过大(如申请2^100000，go中的uint表示不了这么大的数）
2. 申请的数据空间过大，操作系统分配不了这么大
3. 数组长度为负

那么系统的MaxUintptr是多少，可以分配的maxAlloc又是多少呢？

首先，我们可以看到

```golang
# https://github.com/golang/go/blob/go1.15.6/src/runtime/internal/math/math.go#L9
const MaxUintptr = ^uintptr(0)
```

MaxUintptr是对一个uintptr 0值全部取反，根据参考[3]，可知在64位机器上，uintptr也是64位，即其最大值为`0xffffffffffffffff`

那么最大内存呢？

```golang
# https://github.com/golang/go/blob/go1.15.6/src/runtime/malloc.go#L210
_64bit = 1 << (^uintptr(0) >> 63) / 2 // 32位系统为0，64位为1
......
heapAddrBits = (_64bit*(1-sys.GoarchWasm)*(1-sys.GoosDarwin*sys.GoarchArm64))*48 + (1-_64bit+sys.GoarchWasm)*(32-(sys.GoarchMips+sys.GoarchMipsle)) + 33*sys.GoosDarwin*sys.GoarchArm64

maxAlloc = (1 << heapAddrBits) - (1-_64bit)*1
```

同时分析sys.GoarchWasm常量都是运行平台操作系统相关的，在笔者的mac 64机器上定义如下

```golang
const GoarchWasm = 0
const GoosDarwin = 1
const GoarchArm64 = 0
const GoarchMips = 0
const GoarchMipsle = 0
```

我们可以计算得对地址位为

```
heapAddrBits = (1 * ( 1 - 0 ) * ( 1 - 1 * 0 )) *48 + ( 1 - 1 + 0 ) * ( 32 - ( 0 + 0 )) + 33 * 1 * 0 = 48
```

可知一般分配的最大内存为

```
( 1 << 48 ) - ( 1 -1 ) * 1 = 1<<48
```

为什么是48位，是由于硬件和操作系统的限制，在这就不赘述了。

## 结论

通过以上分析，我们可以得到结论，go能够创建的数组长度大小，取决于类型和平台，在不同平台上，值是不同的。

以我们通用的amd64平台来说，一般长度需要满足以下两个条件

1. ```
   sizeof(data type) * len < ^uintptr(0) # 该值为2^64-1
   ```

2. ```
   sizeof(data type) * len < maxAlloc # 该值为 2^48
   ```

## 参考

1. [make原理分析](https://zhuanlan.zhihu.com/p/76524813)
2. [make&new关键字](http://qqailab.cn/2019/10/17/golang-make-new.html)
3. [uintptr定义](https://go.dev/tour/basics/11)

