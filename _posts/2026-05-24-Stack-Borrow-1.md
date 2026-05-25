---
title: Stack-Borrows-1
date: 2026-05-24 22:21:00 +0800
categories: [Rust]
tags: [Stack-Borrows]
---

# Stack Borrows

这篇Post主要分享学习Stack-Borrows的过程和感悟。关于Stack-Borrows的实现和内容，主要从这篇Blog中学到：[Stack-Borrows](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md)。如果有问题和疑问，欢迎指出。

## Reason

几周前，在跟导师讨论时，提到了标准库的一个API: [core::ptr::copy](https://doc.rust-lang.org/beta/core/ptr/fn.copy.html)

```rust
pub const unsafe fn copy<T>(src: *const T, dst: *mut T, count: usize)
```

这个API的safety描述为：

>Behavior is undefined if any of the following conditions are violated:
>
>* src must be valid for reads of count * size_of::<T>() bytes or that number must be 0.
>
>* dst must be valid for writes of count * size_of::<T>() bytes or that number must be 0, and dst must remain valid even when src is read for count * size_of::<T>() bytes. (This means if the memory ranges overlap, the dst pointer must not be invalidated by src reads.)
>
>* Both src and dst must be properly aligned.

其中括号内的描述比较有意思：`This means if the memory ranges overlap, the dst pointer must not be invalidated by src reads.`

当时导师给的解释有点忘记了，但是大概意思是如果指针指向的内存中记录了指向别处的指针，那么copy时会出现意外。

但是当时我不理解 `dst` 为什么会被 `src读` 失效，问了AI后，用一些代码在rustplayground上跑，下面是第一段代码：

```rust
fn main() {
    let mut arr = [10, 20, 30, 40];

    let base_ptr = arr.as_mut_ptr();

    unsafe {
        let dst_slice = &mut arr[1..4];
        let dst = dst_slice.as_mut_ptr();

        let src = base_ptr as *const i32;

        std::ptr::copy(src, dst, 3);
    }

    println!("{:?}", arr);
}
```

Miri给出的报错是:

```text
error: Undefined Behavior: attempting a read access using <385> at alloc201[0x0], but that tag does not exist in the borrow stack for this location
  --> src/main.rs:12:9
   |
12 |         std::ptr::copy(src, dst, 3);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ this error occurs as part of an access at alloc201[0x0..0xc]
   |
   = help: this indicates a potential bug in the program: it performed an invalid operation, but the Stacked Borrows rules it violated are still experimental
   = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md for further information
help: <385> was created by a SharedReadWrite retag at offsets [0x0..0x10]
  --> src/main.rs:4:20
   |
 4 |     let base_ptr = arr.as_mut_ptr();
   |                    ^^^^^^^^^^^^^^^^
help: <385> was later invalidated at offsets [0x0..0x10] by a Unique retag
  --> src/main.rs:7:30
   |
 7 |         let dst_slice = &mut arr[1..4];
   |                              ^^^
```

## Stack-Borrows

在写这篇post的时候，我已经阅读了一些Stack-Borrows的文档，在zulip上也open了一个关于这个API的[讨论](https://rust-lang.zulipchat.com/#narrow/channel/136281-t-opsem/topic/Different.20output.20when.20testing.20std.3A.3Aptr.3A.3Acopy.20by.20miri/with/595183915)，对于这个UB出现的原因，以及“invalidate”的含义，都大概了解了。

首先需要介绍一下Stack-Borrows是什么。Stack-Borrows是一个Aliasing模型。Aliasing就是别名关系，如果一个内存区域，有多个指针或者有多个引用同时指向这个内存，那么可以说他们具有Alias关系。[The Rustonomicon](https://doc.rust-lang.org/nomicon/aliasing.html)和[Rust By Example](https://doc.rust-lang.org/rust-by-example/scope/borrow/alias.html)中有介绍Aliasing的章节。在Rust中，Aliasing主要有下面的要求：
* 可以同时存在多个不可变引用，但是在不可变引用被使用时，不可以存在可变引用的借用，也就是读操作发生时，不允许发生写操作。
* 在某一时刻，只能有至多一个可变引用在使用，不能同时存在不可变引用。即写操作发生时，不能够进行读操作和其他写操作。

回到Stack-Borrows，作为一个Aliasing Model，这个模型可以用来检测一些非法的内存操作，但是规则是由Stack-Borrows制定的。如果要了解Stack-Borrows做了什么，可以先去看[github](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md)上的文档，以及[原论文](https://plv.mpi-sws.org/rustbelt/stacked-borrows/paper.pdf)。

下面来解释一下为什么上面的例子在使用Stack-Borrows时会检测到UB。

Stack-Borrows会在创建新引用或者新的裸指针时，分配出一个唯一性的Tag（通过递增实现唯一性），通过引用创建引用或者指针时，会基于当前Tag创建出新的Tag，这个过程叫重新标记(ReTag)。ReTag的场景有以下几种，每一种场景都有各自ReTag的类型：

1. 普通的赋值操作之后.只把一个引用（& / &mut）或者 Box 赋值给另一个变量，Stack-Borrows就会给新变量来一次 Default 或 TwoPhase 的 Retag。
2. 引用变成裸指针时：写下类似 &mut x as *mut i32 的强制转换时，会进行一次 Raw Retag。
3. 函数入口：每个函数刚开始执行的第一条指令，会对它接收到的所有参数进行一次 FnEntry Retag。
4. 接收函数返回值时：调用函数并传递它返回的引用时，该引用会经历一次 Default Retag。
5. 变量被 Drop 销毁时：当变量离开作用域触发自动清理逻辑时，会把传入的参数当作裸指针，进行一次 Raw Retag 。

Tag具有不同的许可(Permission)，[github](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md#extra-state)上给出了Stack-Borrows中许可的四种分类：Unique(&mut T)，SharedReadWrite(内部可变性类型以及裸指针)，SharedReadOnly(&T)和Disabled(失去读写权限，可以作为标志位，通常由在栈中的Tag进行的读操作引起)。对于一个内存区域，其相关的裸指针以及引用都通过一个栈来记录活跃的Tag，创建一个新的Tag就会push到栈中，在生命周期结束时就会pop出栈(栈中真正存储的应该叫作Item，Tag其实只是地址，一个Item拥有Tag、许可和Protector，Protector用于作为函数参数时保护参数)。但是在进栈时，会根据Tag的Permission有额外的处理：如果一个Unique的Tag进栈，那么在栈中所有已有的Tag都会Pop出栈，因此被称为“invalidate”。

可以推测原因：一个可变引用被创建，那么不允许存在任何不可变引用以及派生。这个规则导致了在第七行创建dst_slice时，base_ptr就不能再被用于内存读取，因为对应的Tag已经不在栈中。这对应了UB的第一行报错: `attempting a read access using <385> at alloc201[0x0], but that tag does not exist in the borrow stack for this location`

但是Stack-Borrows只有上面的设计，在遇到 **Two-Phase-Borrows** 时会出现问题:

```rust
fn two_phase() {
    let mut v = vec![];
    v.push(v.len());
}
```

上面的代码使用刚才提到的设计就会触发UB，因为`v.push()`先运行，Stack-Borrows发现存在一个可变引用的使用，在栈中push一个Unique Tag。但是作为参数,`v.len()`会在`v.push()`调用后继续运行，`v.len()`运行结束之后再回到`v.push()`中。在运行`v.len()`时发现目前有一个可变引用的Tag已经在栈中，而`v.len()`使用的是不可变引用，两者不能同时存在。

为了解决这个问题，Stack-Borrows设计了新的机制。在可变引用创建时，会判断这个可变引用是否是Two-Phase-Borrows：

```rust
pub enum RefKind {
    /// `&mut` and `Box`.
    Unique { two_phase: bool },
    /// `&` with or without interior mutability.
    Shared,
    /// `*mut`/`*const` (raw pointers).
    Raw { mutable: bool },
}

fn reborrow(
    self: &mut MiriInterpContext,
    place: MPlaceTy<Tag>,
    kind: RefKind,
    new_tag: Tag,
    protect: Option<ProtectorKind>,
)
```

如果是用于Two-Phase-Borrows，那么这个Tag进栈时不会将所有的Tag弹出栈，也不会禁止之后的不可变引用，类似Two-Phase-Borrows的Frozen阶段。上面的例子中，`v.len()`重新借用产生的不可变引用可以正常使用。当`v.len()`结束，`v.push()`真正运行时，这个Tag才会将已有的不可变引用弹出栈，不允许通过其他引用进行任何读取，类似Two-Phase-Borrows的Active阶段。

## Tree-Borrows

神奇的是，上述例子使用Tree-Borrows再次运行，并不会检测出UB。

这说明Tree-Borrows的规则相比于Stack-Borrows，更为宽松，Tree-Borrows认为这段代码没有问题。

但是我们还没有回答这个问题：什么情况下会违反“dst pointer must not be invalidated by src reads”

在Tree-Borrows中，可以产生这个场景:

```rust
fn main() {
    let mut arr = std::cell::Cell::new([10u8, 20, 30, 40]);
    let src = &mut arr as *mut _ as *const u8;
    let dst = &mut arr as *mut _ as *mut u8;
    let dst = unsafe { dst.offset(1) };
    unsafe { dst.write(22) };
    unsafe { std::ptr::copy(src, dst, 3) };
    println!("{arr:?}");
}
```

运行Miri会发现UB:

```text
error: Undefined Behavior: write access through <378> at alloc209[0x1] is forbidden
 --> src/main.rs:7:14
  |
7 |     unsafe { std::ptr::copy(src, dst, 3) };
  |              ^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a potential bug in the program: it performed an invalid operation, but the Tree Borrows rules it violated are still experimental
  = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/tree-borrows.md for further information
  = help: the accessed tag <378> has state Frozen which forbids this child write access
help: the accessed tag <378> was created here, in the initial state Reserved (interior mutable)
 --> src/main.rs:4:15
  |
4 |     let dst = &mut arr as *mut _ as *mut u8;
  |               ^^^^^^^^
help: the accessed tag <378> later transitioned to Unique due to a child write access at offsets [0x1..0x2]
 --> src/main.rs:6:14
  |
6 |     unsafe { dst.write(22) };
  |              ^^^^^^^^^^^^^
  = help: this transition corresponds to the first write to a 2-phase borrowed mutable reference
help: the accessed tag <378> later transitioned to Frozen due to a foreign read access at offsets [0x0..0x3]
 --> src/main.rs:7:14
  |
7 |     unsafe { std::ptr::copy(src, dst, 3) };
  |              ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  = help: this transition corresponds to a loss of write permissions

note: some details are omitted, run with `MIRIFLAGS=-Zmiri-backtrace=full` for a verbose backtrace

error: aborting due to 1 previous error
```

可以发现，这个时候检测到的UB是对dst进行写操作被禁止。这跟Tree-Borrows的设计有关系。跟Stack-Borrows中使用栈来记录所有别名操作不同，Tree-Borrows用一个树来记录，每一个起始的引用对应这个内存地址创建出的一个新的分支，派生的引用和指针作为对应分支的节点，将derive的派生关系表示的更合理。

在Tree-Borrows中，当一个可变引用被创建时，不会立刻独占这个内存区域，而是被标记为`Reserved`，这跟Rust的[**Two-phase-borrows**](https://rustc-dev-guide.rust-lang.org/borrow-check/two-phase-borrows.html)很像。只有使用这个引用或者派生出的引用或裸指针进行真正的写操作时，会将这个引用及其自引用的Tag更改为Active，表示对这块内存进行独占式写入。

但是写入之后被来自其他分支的Tag进行读操作(Foreign Read)时，会将这个Active的Tag更改为Frozen，表示这个Tag目前不能用于写操作，因为有读操作正在同时发生，并且不是来自这一个分支的Tag。

## 总结

Stack-Borrows 和Tree-Borrows 都是Aliasing Model，通过各自的规则来检测程序是否满足，如果不满足就认为存在未定义行为(Undefined Behaviour, UB)。但是Tree-Borrows的规则相对更加宽松，能够通过一些被Stack-Borrows的情况。

虽然上面的例子使用`cargo run`也会输出预期的结果，但是存在未定义行为就说明这个程序存在风险，并不是百分百安全。在创建或者使用引用和裸指针时，也要遵守Aliasing Model定义的规则。