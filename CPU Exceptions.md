CPU Exceptions
==============

CPU Exceptions는 유효하지 않은 메모리 주소에 접근하거나 0으로 나누는 등의 다양한 에러 상황에서 발생한다. 이러한 에러 상황에 대처하기 위해, handler 함수를 제공하는 interrupt descriptor table을 설정해야 한다. 이 포스트 마지막에서, 우리의 커널은 breakpoint exceptions를 포착하고, 이후에 정상적인 실행을 재개할 수 있을 것이다.

# Overview

예외는 현재의 명령과 관련하여 무언가 잘못되었음을 알려준다. 예를 들어, CPU는 현재의 명령이 0으로 나누는 연산을 시도한다면 예외를 알린다. 예외가 발생하면, CPU는 현재의 작업을 중단하고 즉시 예외 타입에 따른 특정 예외 처리 함수를 실행한다.

x86에서, 20개의 서로 다른 CPU 예외 타입이 있다. 가장 중요한 것들은:
- Page Fault: page fault는 잘못된 메모리 접근에 대하여 발생한다. 예를 들어, 현재 명령이 unmapped page로부터 읽거나 읽기 전용 page에 쓰려고 시도하는 경우가 있다.
- Invalid Opcode: 이 예외는 현재 명령이 유효하지 않은 경우에 발생한다. 예를 들어, 새로운 SSE instructions를 지원하지 않는 오래된 CPU에서 이를 사용하는 경우가 있다.
- General Protection Fault: 이는 넓은 범위의 원인을 가지는 예외다. 이는 다양한 종료의 접근 위반 시 발생한다. 예를 들어, 사용자 수준 코드에서 관리자 명령을 실행하려고 하는 경우나 configuration registers의 예약된 공간에 쓰려고 하는 경우가 있다.
- Double Fault: 이 예외가 발생하면, CPU는 대응하는 처리 함수를 실행하려고 시도한다. 만약 예외 처리 함수를 실행하는 과정에서 다른 예외가 발생한다면, CPU는 double fault exception을 발생시킨다. 이 예외는 어떤 예외에 대해 등록된 처리 함수가 없는 경우에도 발생한다.
- Triple Fault: CPU가 double fault 처리 함수를 실행하려는 동안 예외가 발생한 경우, 치명적인 triple fault가 발생한다. triple fault는 포착하거나 처리할 수 없다. 대부분의 프로세서들은  스스로를 리셋하거나 운영체제를 다시 부팅하는 방법으로 대처한다.

## The Interrupt Descriptor Table

예외를 포착하고 처리하기 위해서, Interrupt Descritor Table (IDT)를 설정해야 한다. 이 table에서, 각각의 CPU 예외에 대한 처리 함수를 구체화할 수 있다. 하드웨어는 이 table을 직접적으로 사용하고, 따라서 우리는 이미 정해진 형식을 따라야 한다. 각각의 시작은 16-byte 구조를 가져야 한다:
Type: Name: Description
u16: Function Pointer [0:15]: The lower bits of the pointer to the handler function.
u16: GDT selector: Selector of a code segment in the global descriptor table.
u16: Options (see below)
u16: Function Pointer [16:31]: The middle bits of the pointer to the handler function.
u32: Function Pointer [32:63]: The remaining bits of the pointer to the handler function.
u32: Reserved:

option 부분은 다음과 같은 형식을 가진다:
Bits: Name: Description
0-2: Interrupt Stack Table Index: 
3-7: Reserved:
8: 
9-11: 반드시 1:
12: 반드시 0:
13-14: Descriptor Privilege Level (DPL): 이 처리 함수를 실행하기 위해 필요한 최소 관리자 레벨]
15: Present

각각의 예외는 미리 정해진 IDT 인덱스를 가진다. 예를 들어, invalid opcode 예외는 table 인덱스로 6을 가지고 page fault 예외는 table 인덱스로 14를 가진다. 따라서, 하드웨어는 각각의 예외에 대응하는 IDT 시작점을 자동으로 로드할 수 있다.

예외가 발생하면, CPU는 대략적으로 다음을 실행한다:
1. 몇몇 레지스터를 스택에 넣는다. (명령 포인터와 RFLAGS 레지스터를 포함한다.)
2. Interrupt Descriptor Table (IDT)로부터 대응하는 시작점을 읽는다. 예를 들어, page fault가 발생하면, CPU는 14번째 시작점을 읽는다.
3. 시작점이 존재하는지 확인하고 없으면 double fault가 발생한다.
4. ++
5. 구체적인 GDT 선택자를 코드 부분에 로드한다.
6. 특정한 처리 함수로 넘어간다.

4, 5 단계에 대해 걱정할 필요는 없다; 이후 포스트에서 global descriptor table과 hardware interrupts에 대해 공부할 것이다.

# An IDT Type

우리만의 IDT 타입을 만드는 대신, `x86_64` 크레이트의 `InterruptDescriptorTable` 구조체를 사용할 것이다.

```rust
#[repr(C)]
pub struct InterruptDescriptorTable {
    pub divide_by_zero: Entry<HandlerFunc>,
    pub debug: Entry<HandlerFunc>,
    pub non_maskable_interrupt: Entry<HandlerFunc>,
    pub breakpoint: Entry<HandlerFunc>,
    pub overflow: Entry<HandlerFunc>,
    pub bound_range_exceeded: Entry<HandlerFunc>,
    pub invalid_opcode: Entry<HandlerFunc>,
    pub device_not_available: Entry<HandlerFunc>,
    pub double_fault: Entry<HandlerFuncWithErrCode>,
    pub invalid_tss: Entry<HandlerFuncWithErrCode>,
    pub segment_not_present: Entry<HandlerFuncWithErrCode>,
    pub stack_segment_fault: Entry<HandlerFuncWithErrCode>,
    pub general_protection_fault: Entry<HandlerFuncWithErrCode>,
    pub page_fault: Entry<PageFaultHandlerFunc>,
    pub x87_floating_point: Entry<HandlerFunc>,
    pub alignment_check: Entry<HandlerFuncWithErrCode>,
    pub machine_check: Entry<HandlerFunc>,
    pub simd_floating_point: Entry<HandlerFunc>,
    pub virtualization: Entry<HandlerFunc>,
    pub security_exception: Entry<HandlerFuncWithErrCode>,
    // some fields omitted
}
```

필드들은 `idt::Entry<F>` 타입을 가진다. 이는 IDT 시작점의 필드를 나타내는 구조체다. 타입 파라미터 `F`는 예상되는 처리 함수 타입을 정의한다. 어떤 시작점들은 `HandlerFunc`을 필요로 하고 다른 시작점들은 `HandlerFuncWithErrCode`를 필요로 한다. page fault는 그만의 특정한 타입을 가진다: `PageFaultHandlerFunc`.

먼저 `HandlerFunc` 타입을 보자:
```rust
type HandlerFunc = extern "x86-interrupt" fn(_: InterruptStackFrame);
```

이것은 `extern "x86-interrupt" fn` 타입의 타입 별명이다. `extern` 키워드는 foreign calling convention을 가지는 함수를 정의하고 자주 C 코드와 통신한다. (`extern "C" fn`) 그러나 `x86-interrupt` 실행 convection이란 무엇일까?

# The Interrupt Calling Convention

예외들은 함수 실행과 유사하다: CPU는 실행된 함수의 첫 명령으로 이동하여 실행한다. 이후에, CPU는 반환 주소로 돌아와서 부모 함수의 실행을 재개한다.

그러나 예외와 함수 실행의 주된 차이가 있다: 함수 실행은 컴파일러가 삽입한 `call` 명령에 의해 자주 발생하는 반면, 에외는 어떤 명령에서도 발생할 수 있다. 이 차이의 결과를 이해하려면 함수 실행을 더 자세히 분석해봐야 한다.

Calling conventions는 함수 실행의 구체적인 내용을 명시한다. 예를 들어, 어디에 함수 파라미터가 위치하고 결과가 어떻게 반환되는 지 명시한다. x86_64 리눅스에서 C 함수에 다음 규칙이 적용된다.

- 처음 6개의 정수 인자는 `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` 레지스터로 전달된다.
- 추가적인 인자들은 스택으로 전달된다.
- 결과는 `rax`와 `rdx`로 반환된다.

러스트는 C ABI를 따르지 않는다. 따라서 이러한 규칙들은 `extern "C" fn`이라고 명시된 함수에만 적용된다.

## Preserved and Scratch Registers

calling convection은 레지스터를 두 파트로 나눈다: preserved 레지스터와 scratch 레지스터.

preserved 레지스터의 값들을 함수 실행 동안 바뀌면 안된다. 따라서 실행된 함수("callee")가 이 레지스터들의 기존 값들을 함수의 반환 전에 되돌려놓을 수 있다면 이 레지스터들을 overwrite할 수 있다. 그러므로 이 레지스터들은 "callee-saved"라고 불린다. 일반적인 패턴은 이 레지스터들을 함수 시작 시 스택에 저장하고 함수 종료 전에 값들을 되돌려 놓는다.

반대로, 실행된 함수는 아무런 제한 없이 scratch 레지스터들을 overwrite할 수 있다. 만약 caller가 함수의 실행 동안 scratch 레지스터의 값들을 보존하고 싶다면, 함수 실행 전에 백업해야 한다. (스택에 값을 넣는 방식을 사용한다.) 그래서 scratch 레지스터는 caller-saved이다.

x86_64에서, C calling convention은 다음과 같이 preserved 레지스터와 scratch 레지스터를 구체화하고 있다.

preserved 레지스터: `rbp`, `rbx`, `rsp`, `r12`, `r13`, `r14`, `r15`
scratch 레지스터: `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`, `r9`, `r10`, `r11`

컴파일러는 이러한 귳칙을 알고 있고 따라서 이에 따라 코드를 생성한다. 예를 들어, 대부분의 함수는 `push rbp`와 함께 시작한다. (이는 `rbp`를 스택에 저장하는 명령이다.)

## Preserving all Registers

함수 실행과 대조적으로, 예외는 어떤 명령에서도 발생할 수 있다. 대부분의 경우, 컴파일하는 동안에도 생성된 코드가 예외를 발생시킬지 알 수 없다. 예를 들어, 컴파일러는 어떤 명령이 stack overflow 또는 page fault를 발생시킬지 알 수 없다.

언제 예외가 발생하는 지 알 수 없기 때문에, 예외가 발생하기 전에 어떤 레지스터를 백업할 수 없다. 이는 예외 처리 함수는 caller-saved 레지스터와 연관 있는 calling convention을 사용할 수 없음을 의미한다. 대신, 모든 레지스터를 보존하는 calling convention이 필요하다. `x86-interrupt` calling convention은 그러한 calling convention이고 따라서 함수 반환 시 모든 레지스터의 기존 값이 복구되는 것을 보장한다.

이는 함수 시작점에서 모든 레지스터의 값이 스택에 저장됨을 의미하지 않는다. 대신, 컴파일러가 함수에 의해 overwrite된 레지스터를 복구한다. 이 방식을 통해, 몇몇 레지스터만 사용하는 짧은 함수를 위한 효율적인 코드를 생성할 수 있다.

## The Interrupt Stack Frame

(`call` 명령을 사용한) 일반적인 함수 실행에서, CPU는 목표 함수로 넘어가기 전에 반환 주소를 넣는다. (`ret` 명령을 사용한) 함수 반환 시, CPU는 이 반환 주소를 꺼낸 뒤 반환 주소로 넘어간다. 따라서 일반적인 함수 실행의 스택 프레임은 다음과 같다:

++

하지만, 예외와 중단 처리 함수를 위해서는 반환 주소를 넣는 것만으로 충분하지 않다. 왜냐하면 중단 처리 함수는 자주 복잡한 맥락에서 실행되기 때문이다. (stack pointer, CPU flags 등을 고려해야 한다.) 대신, CPU는 중단을 발생시킬 때 다음과 같이 동작한다:

0. Saving the old stack pointer:
1. Aligning the stack pointer:
2. Switching stacks:
3. Pushing the old stack pointer:
4. Pushing and updating the `RFLAGS` register:
5. Pushing the instruction pointer:
6. Pushing an error code:
7. Invoking the interrupt handler:

따라서 중단 스택 프레임은 다음과 같다:

++

`x86_64` 크레이트에서, 중단 스택 프레임은 `InterruptStackFrame` 구조체로 표현된다. 이것은 중단 처리 함수로 `&mut` 형태로 전달되고 예외의 원인에 대한 추가적인 정보를 되찾는데 사용될 수 있다. 몇몇의 예외만 에러 코드를 푸쉬하기 때문에, 구조체는 에러 코드 부분을 가지지 않는다. 이러한 예외들은 추가적인 `error_code` 인자를 가지는 구분된 `HandlerFuncWithErrCode` 함수 타입을 사용한다.

## Behind the Scenes

`x86-interrupt` calling convention은 강력한 추상 개념이고 예외 처리 과정의 대부분의 복잡한 디테일을 숨긴다. 그러나, 가끔은 어떤 일이 발생했는 지 아는 데 유용하다. `x86-interrupt` calling convention을 간단하게 둘러보자:

- Retrieving the arguments:
- Returning using `iretq`:
- Handling the error code:
- Aligning the stack:

# Implementation

이론을 이해했으니 우리 커널에서 CPU exception을 처리해보자. 먼저 `src/interrupts.rs` 내에 새로운 interrupts 모듈을 생성한다. 그리고 새로운 `InterruptDescriptorTable`을 생성하는 `init_idt` 함수를 생성한다.

```rust
// in src/lib.rs

pub mod interrupts;

// in src/interrupts.rs

use x86_64::structures::idt::InterruptDescriptorTable;

pub fn init_idt() {
    let mut idt = InterruptDescriptorTable::new();
}
```

이제 처리 함수를 추가할 수 있다. 먼저 breakpoint exception을 처리하기 위한 함수를 추가하자. breakpoint exception이란 예외 처리를 테스트 하기 위한 완벽한 예외다. 이것의 유일한 목적은 breakpoint 명령인 `int3`이 실행될 때 일시적으로 프로그램을 중단하는 것이다.

breakpoint 예외는 일반적으로 디버거에서 사용된다: 사용자가 breakpoint를 설정하면, 디비거가 대응하는 명령을 `int3` 명령으로 overwrite하고 따라서 CPU는 해당 코드에 도달하면 breakpoint 예외를 발생시킨다. 사용자가 프로그램이 지속되기를 원하면, 디버거는 `int3` 명령을 기존 명령으로 대체하고 프로그램을 재개한다.

우리의 경우, 아무런 명령을 overwrite할 필요가 없다. 대신, breakpoint 명령이 실행되면 메시지를 출력하고 그 후에 다시 프로그램이 재개되면 된다. 단순한 `breakpoint_handler` 함수를 만들고 우리의 IDT에 추가하자:

```rust
// in src/interrupts.rs

use x86_64::structures::idt::{InterruptDescriptorTable, InterruptStackFrame};
use crate::println;

pub fn init_idt() {
    let mut idt = InterruptDescriptorTable::new();
    idt.breakpoint.set_handler_fn(breakpoint_handler);
}

extern "x86-interrupt" fn breakpoint_handler(
    stack_frame: InterruptStackFrame)
{
    println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame);
}
```

우리의 처리 함수는 메시지를 출력하고 interrupt stack frame을 출력한다.

컴파일을 시도하면 다음과 같은 에러가 발생한다.

```
error[E0658]: x86-interrupt ABI is experimental and subject to change (see issue #40180)
  --> src/main.rs:53:1
   |
53 | / extern "x86-interrupt" fn breakpoint_handler(stack_frame: InterruptStackFrame) {
54 | |     println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame);
55 | | }
   | |_^
   |
   = help: add #![feature(abi_x86_interrupt)] to the crate attributes to enable
```

에러가 발생하는 이유는 `x86-interrupt` calling convention이 아직 불안정하기 때문이다. 이를 사용하기 위해서, `lib.rs`의 가장 위에 `#![feature(abi_x86_interrupt)]`를 추가하면 이를 명확하게 사용할 수 있다.

## Loading the IDT

CPU가 우리의 새로운 interrupt descriptor table을 사용하기 위해, lidt 명령을 사용하여 이를 로드해야 한다. `x86_64` 크레이트의 `InterruptDescriptorTable` 구조체는 load 메서드를 제공한다. 시도해보자:

```rust
// in src/interrupts.rs

pub fn init_idt() {
    let mut idt = InterruptDescriptorTable::new();
    idt.breakpoint.set_handler_fn(breakpoint_handler);
    idt.load();
}
```

이제 우리가 컴파일을 시도하면 다음과 같은 에러가 발생한다:

```
error: `idt` does not live long enough
  --> src/interrupts/mod.rs:43:5
   |
43 |     idt.load();
   |     ^^^ does not live long enough
44 | }
   | - borrowed value only lives until here
   |
   = note: borrowed value must be valid for the static lifetime...
```

`load` 메서드는 `&'static self`를 예상하는데, 이는 프로그램의 완전한 런타임을 위해 유효한 참조이다. 이유는 CPU는 이 table에 매 interrupt마다 접근하기 때문이다. 따라서  `'static` 대신 짧은 라이프타임을 사용하는 것이 use-after-free bugs로 이어질 수 있다.

++ 우리의 `idt`는 스택 위에 생성되고, 따라서 `init` 함수 내부에서만 유효하다. 그 후, 스택 메모리는 다른 함수들을 위해 재사용되고, 따라서 CPU는 무작위 스택 메모리를 IDT로 간주한다. 운 좋게, `InterruptDescriptorTable::load` 메서드는 함수 내부에 라이프타임 요구조건을 인코드해두고, 따라서 러스트 컴파일러는 이러한 버그를 컴파일하는 동안 예방할 수 있다.

이 문제를 해결하기 위해, 우리는 `idt`를 `'static` 라이프타임을 가지는 공간에 저장해야 한다. 이를 위해, `Box`를 사용하여 IDT를 힙에 할당하고, 이를 `'static` 참조로 전환한다. 그러나, 우리는 OS 커널을 만들고 있고 따라서 힙을 가지고 있지 않다. (아직)

대안으로, IDT를 `static`으로 저장할 수 있다:

```rust
static IDT: InterruptDescriptorTable = InterruptDescriptorTable::new();

pub fn init_idt() {
    IDT.breakpoint.set_handler_fn(breakpoint_handler);
    IDT.load();
}
```

그러나 여기에도 문제가 있다: statics는 수정할 수 없고, 따라서 우리 `init` 함수로부터 breakpoint 시작점을 수정할 수 없다. 이 문제는 `static mut`를 사용하여 해결할 수 있다:

```rust
static mut IDT: InterruptDescriptorTable = InterruptDescriptorTable::new();

pub fn init_idt() {
    unsafe {
        IDT.breakpoint.set_handler_fn(breakpoint_handler);
        IDT.load();
    }
}
```

이 변형은 에러 없이 컴파일되지만 관용적인 형태와 거리가 멀다. `static mut`는 data races가 발생하기 쉽고 따라서 각각의 접근에 대해 `unsafe` 블록이 필요하다.

### Lazy Statics to the Rescue

다행히, `lazy_static` 매크로가 존재한다. `static`을 컴파일 시 평가하는 대신, 이 매크로는 `static`이 처음 참조되는 시점에 초기화를 진행한다. 따라서, 초기화 블록에서 대부분의 것들을 할 수 있고 런타임 값을 읽을 수도 있다.

우리는 이미 `lazy_static` 크레이트를 가져오고 있다. (VGA text buffer의 추상 개념을 생성할 때) 따라서 우리는 static IDT를 생성하기 위해 `lazy_static!` 매크로를 직접적으로 사용할 수 있다.

```rust
// in src/interrupts.rs

use lazy_static::lazy_static;

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        idt
    };
}

pub fn init_idt() {
    IDT.load();
}
```

어떻게 이 방법은 `unsafe` 블록을 필요로 하지 않는 것일까? `lazy_static!` 매크로는 `unsafe`를 내부적으로 사용하지만, safe 인터페이스 안에서 추상화된다.

## Running it

우리 커널 내에서 예외 처리가 작동하게 하기 위한 마지막 단계는 `main.rs`에서 `init_idt` 함수를 실행하는 것이다. 이를 직접적으로 실행하는 것 대신, `lib.rs`에서 일반적인 `init` 함수를 도입한다.

```rust
// in src/lib.rs

pub fn init() {
    interrupts::init_idt();
}
```

이 함수를 통해, `main.rs`, `lib.rs`와 통함된 테스트 내 서로 다른 `_start` 함수 간에 공유되는 초기화 루틴을 위한 central place를 가진다.

`init`를 실행하기 위해 `main.rs`의 `_start` 함수를 수정하고 breakpoint 예외를 테스트해보자.

```rust
// in src/main.rs

#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    blog_os::init(); // new

    // invoke a breakpoint exception
    x86_64::instructions::interrupts::int3(); // new

    // as before
    #[cfg(test)]
    test_main();

    println!("It did not crash!");
    loop {}
}
```

CPU가 성공적으로 우리의 breakpoint 처리 함수를 실행시킨다. breakpoint 처리 함수는 메시지를 출력하고, `_start` 함수로 되돌아 온다. 이후 `It did not crash!` 메시지가 출력된다.

interrupt stack frame은 예외가 발생한 시점의 명령과 스택 포인터를 보여준다. 이 정보는 예상치 못한 예외를 디버깅하는 데 도움이 된다.

## Adding a Test

```rust
// in src/lib.rs

/// Entry point for `cargo test`
#[cfg(test)]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    init();      // new
    test_main();
    loop {}
}
```

```rust
// in src/interrupts.rs

#[test_case]
fn test_breakpoint_exception() {
    // invoke a breakpoint exception
    x86_64::instructions::interrupts::int3();
}
```

```
blog_os::interrupts::test_breakpoint_exception...	[ok]
```

# Too much Magic?