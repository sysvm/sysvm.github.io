# Two Dark Magic Tricks of Golang

## 魔法1：调用runtime中的私有函数

按照 Go 的编译约定，代码包内以小写字母开头的函数、变量是私有的：

```go
package test

// 私有
func abs() {}

// 公共
func Abs(){}
```

对于test包中abs函数只能在包内调用，而Abs函数却可以在其它包中导入后使用。

私有变量、方法的意义在于封装：控制内部数据、保证外部交互的一致性。

这样既能促进系统运行的可靠性，也能减少使用者的信息负载。

这样的规定对设计、封装良好的包是友好的，但并不是每个人都有这样的能力，另外对于一些特殊的函数，如：`runtime中的memmove函数`，在有些场景下，确实是需要的。

因此Go在程序链接阶段给开发者打开了一扇窗，即可以通过 `go:linkname`指令来链接包内的私有函数。

### memmove

以 [memmove](https://github.com/golang/go/blob/1724077b789ad92972ab1ac03788389645306cbb/src/runtime/stubs.go#L111) 为例， 如下：

```go
func memmove(to, from unsafe.Pointer, n uintptr)
```

memmove作为runtime中的私有函数，用于任意数据之间的内存拷贝，无视类型信息，直接操作内存，这样的操作在Go中虽然是不提倡的，但是用好了却也是一把利刃。

新建一个go文件，如runtime.go，并加上如下内容：

```go
//go:noescape
//go:linkname memmove runtime.memmove
//goland:noinspection GoUnusedParameter
func memmove(to unsafe.Pointer, from unsafe.Pointer, n uintptr)
```

把视角放到 `go:linkname`指令上，该指令接受两个参数：

* memmove：当前函数名称；
* runtime.memmove：对应链接的函数的路径，报名+函数名。

这样，编译器在做链接时就会将当前的memmove函数链接到runtime中的memmove函数， 我们就能使用该函数了。

在平常写代码的时候，我们经常性地需要拷贝字节切片、字符串之间的数据。比如将数据从切片1拷贝到切片2，使用memmove代码如下：

```go
// runtime.go
type GoSlice struct{
    Ptr unsafe.Pointer
    Len int
    Cap int
}

// runtime_test.go
func Test_memmove(t *testing.T){
    src :=[]byte{1,2,3,4,5,6}
    dest :=make([]byte,10,10)

    spew.Dump(src)
    spew.Dump(dest)

    srcp :=(*GoSlice)(unsafe.Pointer(&src))
    destp :=(*GoSlice)(unsafe.Pointer(&dest))

    memmove(destp.Ptr, srcp.Ptr, unsafe.Sizeof(byte(0))*6)

    spew.Dump(src)
    spew.Dump(dest)
}
```

字节切片([]byte)在内存中的形态如GoSlice结构体来所示，Len、Cap分别表示切片长度、容量，字段 Ptr 指向真实的字节数据。

将两个切片的数据指针以及拷贝长度作为参数传入memmove，数据就能从src拷贝到dest。运行结果如下：

```go
===RUN   Test_memmove
# 拷贝之前
([]uint8)(len=6 cap=6){
00000000010203040506|......|
}
([]uint8)(len=10 cap=10){
0000000000000000000000000000|..........|
}
# 拷贝之后
([]uint8)(len=6 cap=6){
00000000010203040506|......|
}
([]uint8)(len=10 cap=10){
0000000001020304050600000000|..........|
```

显然，对于切片之间的数据拷贝，标准库提供的copy函数要更加方便一些：

```go
func Test_copy(t *testing.T){
    src :=[]byte{1,2,3,4,5,6}
    dest :=make([]byte,10,10)

    spew.Dump(src)
    spew.Dump(dest)

    copy(dest, src)

    spew.Dump(src)
    spew.Dump(dest)
}
```

这样也能达到一样的效果，memmove更加适合字符串(string)和数组切片之间的数据拷贝场景，如下：

```go
// runtime.go
type GoString struct{
    Ptr unsafe.Pointer
    Len int
}

// runtime_test.go
func Test_memmove(t *testing.T){
    str :="pedro"
// 注意：这里的len不能为0，否则数据没有分配，就无法复制
    data :=make([]byte,10,10)
    spew.Dump(str)
    spew.Dump(data)

    memmove((*GoSlice)(unsafe.Pointer(&data)).Ptr,(*GoString)(unsafe.Pointer(&str)).Ptr,
        unsafe.Sizeof(byte(0))*5)
    spew.Dump(str)
    spew.Dump(data)
}
```

类似地，GoString是字符串在内存中的表达形态，通过memmove函数就能快速的将字符数据从 字符串拷贝到切片，反之亦然，运行结果如下：

```go
# 拷贝之前
(string)(len=5)"pedro"
([]uint8)(len=10 cap=10){
0000000000000000000000000000|..........|
}
# 拷贝之后
(string)(len=5)"pedro"
([]uint8)(len=10 cap=10){
00000000706564726f 0000000000|pedro.....|
}
```

### growslice

切片是Go中最常用的数据结构之一，对于切片扩容，Go只提供了append函数来隐式的扩容，但内部是通过调用runtime中的[growslice](https://github.com/golang/go/blob/1724077b789ad92972ab1ac03788389645306cbb/src/runtime/slice.go#L166)函数来实现的：

```go
func growslice(et *_type, old slice,capint) slice
```

growslice函数接受3个参数：

* et：切片容器中的数据类型，如int，_type可以表示Go中的任意类型；
* old：旧切片；
* cap：扩容后的切片容量。

扩容成功后，返回新的切片。

同样地，使用 `go:linkname`来链接runtime中的growslice函数，如下：

```go
// runtime.go
type GoType struct{
    Size       uintptr
    PtrData    uintptr
    Hash       uint32
    Flags      uint8
    Align      uint8
    FieldAlign uint8
    KindFlags  uint8
    Traits     unsafe.Pointer
    GCData     *byte
    Str        int32
    PtrToSelf  int32
}

// GoEface 本质是 interface
type GoEface struct{
    Type  *GoType
    Value unsafe.Pointer
}

//go:linkname growslice runtime.growslice
//goland:noinspection GoUnusedParameter
func growslice(et *GoType, old GoSlice,capint) GoSlice
```

growslice函数的第一个参数et实际是Go对所有类型的一个抽象数据结构——GoType。

这里引入了Go语言实现机制中的两个重要数据结构：

* GoEface：empty interface，即interface{}，空接口；
* GoType：Go类型定义数据结构，可用于表示任意类型。

关于GoEface、GoIface、GoType、GoItab都是Go语言实现的核心数据结构，这里的内容很多， 感兴趣的可以参考[这里](https://zhuanlan.zhihu.com/p/427838620) 。

这样，我们就能通过调用growslice函数来对切片进行手动扩容了，如下：

```go
// runtime.go
func UnpackType(t reflect.Type)*GoType {
    return(*GoType)((*GoEface)(unsafe.Pointer(&t)).Value)
}

// runtime_test.go
func Test_growslice(t *testing.T){
    assert := assert.New(t)

    var typeByte =UnpackType(reflect.TypeOf(byte(0)))

    spew.Dump(typeByte)

    dest :=make([]byte,0,10)

    assert.Equal(len(dest),0)
    assert.Equal(cap(dest),10)

    ds :=(*GoSlice)(unsafe.Pointer(&dest))
    *ds =growslice(typeByte,*ds,100)

    assert.Equal(len(dest),0)
    assert.Equal(cap(dest),112)
}
```

由于growslice的参数et类型在runtime中不可见，我们重新定义了GoType来表示， 并且通过反射的机制来拿到字节切片中的GoType，然后调用growslice完成扩容工作。

运行程序：

```go
---PASS:Test_growslice(0.00s)
PASS
```

注意一个点，growslice传入的cap参数是100，但是最后的扩容结果却是112， 这个是因为 growslice会做一个roundupsize处理，感兴趣的同学可以参考[这里](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/) 。

## 魔法 2：调用C/汇编函数

下面，我们再来看Go的另外一个更加有趣的黑魔法。

### cgo

通过cgo，我们可以很方便地在Go中调用C代码，如下：

```go
/*
#include <stdio.h>
#include <unistd.h>

static void* Sbrk(int size) {
    void *r = sbrk(size);
    if(r == (void *)-1 {
        return NULL;
    }
    return r;
}
*/
import"C"

import (
    "fmt"
)

func main(){
    mem := C.Sbrk(C.int(100))
    defer C.free(mem)
    fmt.Println(mem)
}
```

运行程序，会得到如下输出：

```go
0xba00000
```

cgo是Go与C之间的桥梁，让Go可以享受C语言强大的系统编程能力，比如这里的sbrk会直接向进程申请一段内存，而这段内存是不受Go GC的影响的，因此我们必须手动地释放(free)掉它。

在一些特殊场景，比如全局缓存，为了避免数据被GC掉而导致缓存失效，那么可以尝试这样使用。

当然，这还不够tricky，别忘了，C语言是可以直接内联汇编的，同样地，我们也可以在Go中内联汇编试试，如下：

```go
/*
#include <stdio.h>

static int Add(int i, int j)
{
    int res = 0;
    __asm__ ("add %1, %2"
        : "=r" (res)
        : "r" (i), "0" (j)
    );
    return res;
}
*/
import"C"
import (
    "fmt"
)

func main(){
    r := C.Add(C.int(2022), C.int(18))
    fmt.Println(r)
}
```

运行程序，可以得到如下输出：

```go
2040
```

cgo虽然给了我们一座桥梁，但付出的代价也不小，具体的缺点可以参考[这里](https://cloud.tencent.com/developer/article/1650525)。

### 对cgo感兴趣的同学可以参考[这里](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-03-cgo-types.html) 。

### 汇编

#### isspace

那么有没有一种方式可以回避掉cgo的缺点，答案自然是可以的。

这个方式其实很容易想到：不使用cgo，而是使用plan9，也就是Go支持的汇编语言。

当然我们不是直接去写汇编，而是将C编译成汇编，然后再转化成plan9与.go代码一起编译。

编译的过程如下图所示：

![](https://docimg8.docs.qq.com/image/kNeC741bkwoEAKjQdrExpg.png?w=1280&h=242.6455519329297)

而且C本身就是汇编的高级抽象，作为目前最强劲性能的存在，这种方式不仅回避了cgo的性能问题， 反而将程序性能提高了。过程如下：

首先，我们定义一个简单的C语言函数isspace(判断字符为空)：

```C
// ./inner/op.h
#ifndef OP_H
#define OP_H

char isspace(char ch);

// ./inner/op.c
#include "op.h"

char isspace(char ch){
    return ch ==' '|| ch =='\r'|| ch =='\n'| ch =='\t';
}
```

然后，使用clang将其编译为汇编(注意：是clang)：

```bash
$ clang -mno-red-zone -fno-asynchronous-unwind-tables -fno-builtin -fno-exceptions \
-fno-rtti -fno-stack-protector -nostdlib -O3-msse4 -mavx -mno-avx2 -DUSE_AVX=1 \
-DUSE_AVX2=0-S./inner/*.c
```

编译成功后，会在inner文件夹下生成一个op.s汇编文件，大致如下：

```bash
.section    __TEXT,__text,regular,pure_instructions
.build_version macos,11,0
.globl  _isspace                        ## -- Begin function isspace
.p2align    4,0x90
_isspace:                               ## @isspace
## %bb.0:
    pushq   %rbp
    movq    %rsp,%rbp
    movb    $1,%al
    cmpb    $13,%dil
    je  LBB0_3
```

clang默认生成的汇编是AT&T格式的，这种汇编Go是无法编译的(gccgo除外)，因此这里有一步转换工作。

负责将AT&T汇编转化成plan9汇编，而二者之间的语法差异其实是比较大的，因此这里借助一个转换[asm2asm 工具](https://github.com/chenzhuoyu/asm2asm) 来完成。

将asm2asm clone到本地，然后运行：

```bash
$ git clone https://github.com/chenzhuoyu/asm2asm
$ ./tools/asm2asm.py ./op.s ./inner/op.s
```

执行后，会报错。原因在于，Go对于plan9汇编文件需要一个对应的.go声明文件来对应。

我们在 ./inner/op.h 文件中定义了isspace函数，因此需要新建一个同名的op.go文件来声明这个函数：

```go
//go:nosplit
//go:noescape
//goland:noinspection GoUnusedParameter
func __isspace(ch byte)(ret byte)
```

然后再次运行asm2asm工具来生成汇编：

```bash
$ ./tools/asm2asm.py ./op.s ./inner/op.s

$ tree .

.
|__ inner
||__  op.c
||__ op.h
||__ op.s
|__ op.go
|__ op.s
|__ op_subr.go
```

asm2asm会生成两个文件：op.s和op_subr.go：

* op.s：翻译而来的plan9汇编文件；
* op_subr.go：函数调用辅助文件。

生成后，op.go中的 `__isspace`函数就能顺利的链接上对应的汇编代码，并运行，如下：

```go
func Test___isspace(t *testing.T) {
	type args struct {
		ch byte
	}
	tests := []struct {
		name    string
		args    args
		wantRet byte
	}{
		{
			name:    "false",
			args:    args{ch: '0'},
			wantRet: 0,
		},
		{
			name:    "true",
			args:    args{ch: '\n'},
			wantRet: 1,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if gotRet := __isspace(tt.args.ch); gotRet != tt.wantRet {
				t.Errorf("__isspace() = %v, want %v", gotRet, tt.wantRet)
			}
		})
	}
}

// output
=== RUN   Test___isspace
=== RUN   Test___isspace/false
=== RUN   Test___isspace/true
--- PASS: Test___isspace (0.00s)
--- PASS: Test___isspace/false(0.00s)
--- PASS: Test___isspace/true(0.00s)
PASS
```

__isspace顺利运行，并通过了单测。

#### u32toa_small

一个 `isspace`函数有些简单，无法完全发挥出汇编的能力，下面我们来看一个稍微复杂一点的例子：将整数转化为字符串。

在Go中，整数转化为字符串的方式有多种，比如说：`strconv.Itoa`函数。

这里，我选择用C来写一个简单的整数转字符串的函数：`u32toa_small`，然后将其编译为汇编代码供Go调用，并看看二者之间的性能差异。

`u32toa_small`的实现也比较简单，使用查表法(strconv.Itoa使用的也是这种方法)，如下：

```C
#include "op.h"

staticconstchar Digits[200]={
    '0','0','0','1','0','2','0','3','0','4','0','5','0','6','0','7','0','8','0','9',
    '1','0','1','1','1','2','1','3','1','4','1','5','1','6','1','7','1','8','1','9',
    '2','0','2','1','2','2','2','3','2','4','2','5','2','6','2','7','2','8','2','9',
    '3','0','3','1','3','2','3','3','3','4','3','5','3','6','3','7','3','8','3','9',
    '4','0','4','1','4','2','4','3','4','4','4','5','4','6','4','7','4','8','4','9',
    '5','0','5','1','5','2','5','3','5','4','5','5','5','6','5','7','5','8','5','9',
    '6','0','6','1','6','2','6','3','6','4','6','5','6','6','6','7','6','8','6','9',
    '7','0','7','1','7','2','7','3','7','4','7','5','7','6','7','7','7','8','7','9',
    '8','0','8','1','8','2','8','3','8','4','8','5','8','6','8','7','8','8','8','9',
    '9','0','9','1','9','2','9','3','9','4','9','5','9','6','9','7','9','8','9','9',
};

// < 10000
int u32toa_small(char*out,uint32_t val) {
    int      n  =0;
    uint32_t d1 =(val /100)<<1;
    uint32_t d2 =(val %100)<<1;

    /* 1000-th digit */
    if(val >=1000){
            out[n++]= Digits[d1];
    }

    /* 100-th digit */
    if(val >=100){
            out[n++]= Digits[d1 +1];
    }

    /* 10-th digit */
    if(val >=10){
            out[n++]= Digits[d2];
    }

    /* last digit */
        out[n++]= Digits[d2 +1];
    return n;
}
```

然后在op.go中加入对应的 `__u32toa_small`函数：

```go
// < 10000
//go:nosplit
//go:noescape
//goland:noinspection GoUnusedParameter
func __u32toa_small(out *byte, val uint32)(ret int)
```

使用clang重新编译op.c文件，并用asm2asm工具来生成对应的汇编代码(节选部分)：

```go
_u32toa_small:
    BYTE $0x55 // pushq        %rbp
    WORD $0x8948;BYTE $0xe5 // movq         %rsp, %rbp
    MOVLSI,AX
    IMUL3Q $1374389535,AX,AX
    SHRQ $37,AX
    LEAQ0(AX)(AX*1),DX
    WORD $0xc06b;BYTE $0x64 // imull        $100, %eax, %eax
    MOVLSI,CX
    SUBLAX,CX
    ADDQCX,CX
    CMPLSI, $1000
    JBLBB1_2
    LONG $0x60058d48;WORD $0x0000;BYTE $0x00 // leaq $96(%rip), %rax  /* _Digits(%rip) */
    MOVB0(DX)(AX*1),AX
    MOVBAX,0(DI)
    MOVL $1,AX
    JMPLBB1_3
```

然后在 Go 中调用该函数：

```go
func Test___u32toa_small(t *testing.T) {
	var buf [32]byte
	type args struct {
		out *byte
		val uint32
	}
	tests := []struct {
		name    string
		args    args
		wantRet int
	}{
		{
			name: "9999",
			args: args{
				out: &buf[0],
				val: 9999,
			},
			wantRet: 4,
		},
		{
			name: "1234",
			args: args{
				out: &buf[0],
				val: 1234,
			},
			wantRet: 4,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := __u32toa_small(tt.args.out, tt.args.val)
			assert.Equalf(t, tt.wantRet, got, "__u32toa_small(%v, %v)", tt.args.out, tt.args.val)
			assert.Equalf(t, tt.name, string(buf[:tt.wantRet]), "ret string must equal name")
		})
	}
}
```

测试成功，`__u32toa_small`函数不仅成功运行，而且通过了测试。

最后，我们来做一个性能跑分看看 `__u32toa_small`和 `strconv.Itoa`之间的性能差异：

```go
func BenchmarkGoConv(b *testing.B) {
    val :=int(rand.Int31()%10000)
    b.ResetTimer()
    for n :=0; n < b.N; n++{
        strconv.Itoa(val)
    }
}

func BenchmarkFastConv(b *testing.B) {
    var buf [32]byte
    val :=uint32(rand.Int31()%10000)
    b.ResetTimer()
    for n :=0; n < b.N; n++{
        __u32toa_small(&buf[0], val)
    }
}
```

使用 `go test -bench`运行这两个性能测试函数，结果如下：

```bash
BenchmarkGoConv
BenchmarkGoConv-126074078219.52 ns/op

BenchmarkFastConv
BenchmarkFastConv-121229459249.455 ns/op
```

从结果中，可以明显看出 `__u32toa_small`优于Itoa，大概有一倍的提升。

## 总结

至此，Go的两个黑魔法技巧已经介绍完毕了，感兴趣的同学可以自己实践看看。

Go 的黑魔法一定程度上都使用了unsafe的能力，这也是Go不提倡的，当然使用unsafe其实就和普通的C代码编写一样，因此也无需有太强的心理负担。

实际上，上述的两种方法都被[sonic](https://github.com/bytedance/sonic)用在了生产环境上，而且带来的很大的性能提升，节约大量资源。

因此，当Go现有的标准库无法满足你的需求时，不要受到语言本身的限制，而是用虽然少见但有效的方式去解决它。

希望上面的两个黑魔法能带你对Go不一样的认识。

## 参考资料

* [https://github.com/bytedance/sonic](https://github.com/bytedance/sonic)
* [https://juejin.cn/post/6974581261192921095](https://juejin.cn/post/6974581261192921095)
* [https://www.keil.com/support/man/docs/armclang_intro/armclang_intro_ddx1471430827125.htm](https://www.keil.com/support/man/docs/armclang_intro/armclang_intro_ddx1471430827125.htm)
* [https://karthikkaranth.me/blog/calling-c-code-from-go/](https://karthikkaranth.me/blog/calling-c-code-from-go/)
* [https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-03-cgo-types.html](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-03-cgo-types.html)
* [https://blog.csdn.net/lastsweetop/article/details/78830772](https://blog.csdn.net/lastsweetop/article/details/78830772)
