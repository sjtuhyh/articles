图解Go之内存对齐

参考：
   PPT: https://docs.google.com/presentation/d/1XUA8WfgTHCF_8XdfPEuNvs-WZ0DshFHKFEEqHRd3Tzg/edit#slide=id.g812812c0e8_1_106
   
   http://blog.newbmiao.com/slides/%E5%9B%BE%E8%A7%A3Go%E4%B9%8B%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90.pdf
   
   https://wizardforcel.gitbooks.io/network-basic/content/0.html
   
- 了解内存对齐的收益
- 为什么要对齐
- 怎么对齐：
	- 数据结构对齐
	- 内存地址对齐
- 64位字的安全访问保证（32位平台）

了解内存对齐的收益
- 提高代码平台兼容性
- 优化数据对内存的使用
- 避免一些内存不对齐带来的坑
- 有助于一些源码的阅读

（系统、编译器、语言做了很多 程序员大部分时间都不可见）

为什么要对齐
| 位bit | 计算机内部数据存储的最小单位 |
| 字节byte | 计算机数据处理的基本单位 |
| 机器字machine word | 计算机用来一次性处理事务的一个固定长度|
![image](https://user-images.githubusercontent.com/34932312/98930033-09927200-2517-11eb-9567-00c716ea6df4.png)

1. 平台原因（移植原因）
不是所有硬件平台都可以访问任意地址上的任意数据的；某些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。
2. 性能原因
数据结构应该尽可能在自然边界上对齐。原因在于，为了访问未对齐的内存，处理器需要作两次内存访问；而对齐的内存访问仅需要一次访问。


数据结构对齐 大小保证（size guarantee）
type   							size in bytes
byte，uint8，int8					1
uint16，int16						2
uint32，int32，float32				4
uint64，int64，float64，complex64	8
complex128							16
struct{}，[0]t{}						0

数据结构对齐 对齐保证（align guarantee）
type 						alignment guarantee
bool,byte，uint8，int8				1
uint16，int16						2
uint32，int32						4
float32,complex64					4
arrays 						由其元素类型决定
structs						由其字段类型决定（最大，max is machine word） 
other types					一个机器字（machine word）的大小


![image](https://user-images.githubusercontent.com/34932312/98935596-b91f1280-251e-11eb-87ba-c121737d7455.png)
64位系统最大按8字节对齐

数据结构对齐工具
![image](https://user-images.githubusercontent.com/34932312/98935772-fbe0ea80-251e-11eb-80a8-b7c80c1cc2b3.png)

![image](https://user-images.githubusercontent.com/34932312/98936510-05b71d80-2520-11eb-822b-7ddbc04f560b.png)


数据结构对齐举例：
type Ag struct {
	arr [2]int8 //2
	bl bool  // 1 padding 5
	sl []int16  // 24
	ptr *int64  // 8
	st struct {  // 16
		str string 
	}
	m map[string]int16
	i interface{}
}


不同数据类型底层数据结构
```
// reflect/value.go
// stringHeader is a safe version of StringHeader used within this package.
type stringHeader struct {
	Data uintptr  
	Len  int
}

// SliceHeader is the runtime representation of a slice.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```
string和slice底层都是data len cap的数据结构，只不过string不需要cap
Data是一个指针，或者说是一个数据地址
8+8=16
slice 多加个8 = 24

```
// runtime/runtime2.go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```
interface有两种，一种是接口类型，一种是空接口类型，空接口类型没有实现方法的契约  
2个指针，所以大小是16


```
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```
map 类型占8个字节，是一个指向 map 结构的指针


数据结构对齐  举个例子：final zero field
```
package main

import (
	"fmt"
	"unsafe"
)

type T1 struct {
	a struct{}
	x int64
}

type T2 struct {
	x int64
	a struct{}
}

func main() {
	a1 := T1{}
	a2 := T2{}
	fmt.Printf("zero size struct{} of T1 size: %d, T2 (as final field) size : %d \n", unsafe.Sizeof(a1), unsafe.Sizeof(a2))
}
```
ret：zero size struct{} of T1 size: 8, T2 (as final field) size : 16 

64位系统T2比T1多出了8个字节，为什么会做这个padding？如果需要对这个0字节的数据做个引用，比如T2.a的时候，需要一块内存去承载这个数据。

源码里比如waitgroup nocopy都会放在struct第一个字段 从而不占用字节


数据结构对齐 重排优化（粗暴方式-按对齐值的递减来重排成员）
![image](https://user-images.githubusercontent.com/34932312/99236825-a6fce700-2832-11eb-8c66-6a15bb7acfb7.png)


内存你地址对齐
	计算机结构可能会要求内存地址进行对齐；也就是说，一个变量的地址是一个因子的倍数，也就是该变量的类型是对齐值。
	函数Alignof接受一个表示任何类型变量的表达式作为参数，并以字节为单位返回变量（类型）的对齐值。对于变量x：
	```
	uintptr(unsafe.Pointer(&x)) % unsafe.Alignof(x) == 0
	```

	举例：
	```
	// A WaitGroup waits for a collection of goroutines to finish.
	// The main goroutine calls Add to set the number of
	// goroutines to wait for. Then each of the goroutines
	// runs and calls Done when finished. At the same time,
	// Wait can be used to block until all goroutines have finished.
	//
	// A WaitGroup must not be copied after first use.
	type WaitGroup struct {
		noCopy noCopy

		// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
		// 64-bit atomic operations require 64-bit alignment, but 32-bit
		// compilers do not ensure it. So we allocate 12 bytes and then use
		// the aligned 8 bytes in them as state, and the other 4 as storage
		// for the sema.
		state1 [3]uint32
	}

	// state returns pointers to the state and sema fields stored within wg.state1.
	func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
		if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
			return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
		} else {
			return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
		}
	}
	```
	https://github.com/talkgo/night/issues/588
	18页底下有描述：首先在64位系统和32位系统上，uint32能保证是4bytes对齐, 即state1地址是4N: uintptr(unsafe.Pointer(&wg.state1))%4 == 0
	而为保证8位对齐，我们只需要判断state1地址是否为8的倍数 如果是（N为偶数），那前8bytes就是64位对齐	否则（N为奇数），那后8bytes是64位对齐


为什么是[3]uint32,不是[12]byte？
不能内存对齐，无法服务于后面的原子操作

64位字的安全访问保证（32位系统）
	在x86-32上，64位函数使用Pentium MMX之前不存在的指令。
	在非linux ARM上，64位函数使用ARMv6k内核之前不可用的指令。

	在ARM, x86-32和32位MIPS上，调用方有责任安排对原子访问的64位字的64位对齐。变量或分配的结果、数组或切片中的第一个字（word）可以依赖当做是64位对齐的。

64位字的安全访问保证 Why？
![image](https://user-images.githubusercontent.com/34932312/99246968-25f91c00-2841-11eb-8557-3d491c948bb0.png)

64位字的安全访问保证 How？
![image](https://user-images.githubusercontent.com/34932312/99247327-c5b6aa00-2841-11eb-9334-636cd3939c18.png)


一些源码中的🌰
runtime/runtime2.go
```
type p struct {
	...

	_ uint32 // Alignment for atomic fields below

	// The when field of the first entry on the timer heap.
	// This is updated using atomic functions.
	// This is 0 if the timer heap is empty.
	timer0When uint64

	...

	pad cpu.CacheLinePad
}
```
GMP中的管理groutine本地队列的上下文p中，记录计时器运行时长的uint64，需要保证32位系统上也是8byte对齐（原子操作）

总结
● 内存对齐是为了cpu更高效访问内存中数据
● 结构体对齐依赖类型的大小保证和对齐保证
● 地址对齐保证是：如果类型 t 的对齐保证是 n，那么类型 t 的每个值的地址在运行时必须是 n 的倍数。
● struct内字段如果填充过多，可以尝试重排，使字段排列更紧密，减少内存浪费
● 零大小字段要避免作为struct最后一个字段，会有内存浪 费
● 32位系统上对64位字的原子访问要保证其是8bytes对齐的；当然如果不必要的 话，还是用加锁（mutex）的方式更清晰简单


