---
title: Creating a programming language that can't be interacted with
published: true
---

I recently participated in the [4th installment of langjam](https://github.com/langjam/jam0004), a competition that makes its competitors create a programming language over a period of 48 hours. If that sounds like less time than it should take to make a good programming language, you'd be right, but I reckon that's not really the point of the competition.

Anyway, I learned that this competition existed about half-way through the first day. I decided to look up the prompt which was ```the sound(ness) of one hand typing```. I thought of the old adage which relates to the clapping of a single hand, and got to thinking about ways to design a programming language that was missing half of it, in such a way as to preclude its use. Moreover, we will answer the question "how shit can we make a programming language, before the language becomes completely unusable?"

The result is write-only-wanguage, or 'wow'. Available on github [here](https://github.com/Patrick-Poitras/write-only-wanguage).

# Seizing the means of non-interaction.

If you want to remove half of "read/write", you can choose to either have a read-only language, which sounds boring, or a write-only language. What could a write-only language be? Someone must have made one before, but I didn't bother to check. Can I design a language so absolutely shit that my write-only language is the world's shittest write-only language?

I started writing a parser that would take in data files, and have no further input. But then, isn't that what all compilers do? No, a write-only language would have to have no means of input, no reading of files, or loaded values. Let's push this further and isolate it from the outside world entirely. It has no input, no output. That's the rules.

We're going to allow internal reads and comparisons internally, despite being technically reading, because I decided it was OK, and I'm making up the rules to end up with the most entertaining result.

How do we design a language that can't be interacted with? It can't be compiled, since the compiled code is an output, and it can't really be interpreted since the interpreter needs an input.

But do we **_really_** need an input for an interpreted language?

If we take some inspiration from virtual machines, we can have a state machine that modifies its own state starting from a hardcoded value and proceeding from there. As such, the user does not pass in the input and our interpreter still runs. Magnificent!


## A crappy state machine: designed for jankiness

We're already designing a language with no means of interaction. How could we make this worse? Let's go to a throwback of old. Let's imagine we have a really crappy computer as the basis of our state machine.

Our state machine has an accumulator (ACC, 64-bit), an instruction register (INS, 16-bit). The choice of a 16-bit non-extendable instruction set really increases the jank. It has 256 unsigned 64-bit integers as memory, which we will call "heap". That's it. No stack pointer. No counter. No flags. You have to manually change the instruction register if you want to make it do things.

I'm not certain of this, but I think that not even the worst retrocomputers would be this bad. They had to actually sell units, after all, so we're trailblazing here. This is groundbreaking fundamental research!

Here is what most of the logic flow looks like.

```rust
// Let us define our static registers
static ACC: AtomicU64 = AtomicU64::new(0);
static INS: AtomicU16 = AtomicU16::new(0);

fn main() {
    eval_loop();
}

fn eval_loop() {
    // Here we initialize our main source of memory
    let mut heap: [u64; 256] = [0; 256];
    // Main loop
    loop {
        let ins = INS.load(Ordering::SeqCst);
        INS.store(0, Ordering::SeqCst);

        // Function dispatch on instruction
        let cont: Continuation = match ins {
            0xFFFF => terminate(),
            0x0000 => processor_idle_sleep(),
            // {... more commands ...}
            _ => break,
        };
        match cont {
            Continuation::Stop => break,
            Continuation::Continue => continue,
        };
    }
}
```

The program consists of a single loop that does the following in order:

-   Fetches INS from the atomic static variable, then sets INS = 0;
-   Takes INS, matches it to a function, and then calls the function.
-   The function returns a Continuation enum, which is either Stop or Continue
-   If stop, then break the loop and terminate the program.

ACC starts off at 0, as does INS. The 0 instruction is a no-op. As such the program will loop forever and you have to kill the program to terminate it. Great language!

### Functions

In order to simplify the previous code, I have omitted the functions. The whole list is available on github. Mainly, the functionality is grouped into 4 categories:

| Instruction OPCode Range | Functionality Group                                         |
|------------------------ |----------------------------------------------------------- |
| 0x0000, 0xFFFF           | Sleeps for 10ms, terminates program respectively            |
| 0xA000&#x2026;0xA8FF     | Operations that modify ACC                                  |
| 0xAA00&#x2026;0xAB00     | Operations that write ACC to the heap                       |
| 0xBA00&#x2026;0xBCFF     | Operations that write to INS                                |
| 0xC000&#x2026;0xC3FF     | Operations that write to INS, depending on the value of ACC |

INS is a single value, and does not allow for multiple instruction calls to be queued. As such, it would be seemingly impossible for any meaningful work to be performed.

For instance, let's say we want to zero the ACC. One way this could happen is if INS happened to equal 0xBA04. This is the op-code for "load the value at heap memory address 04 into INS". At address 0x04, if the value 0xA000 was written, it would have the effect of loading A000 into INS, which would then zero the ACC.

Let's work through the control flow here.

| Step | Place in code                     | heap[0x04] | ACC | INS    |
|---- |--------------------------------- |---------- |--- |------ |
| 0    | Start of loop                     | 0xA000     | any | 0xBA04 |
| 1    | Dispatch -> set_ins()   | 0xA000     | any | 0      |
| 2    | After set_ins()         | 0xA000     | any | 0xA000 |
| 3    | Start of loop                     | 0xA000     | any | 0xA000 |
| 4    | Dispatch -> zero_acc()  | 0xA000     | any | 0      |
| 5    | After zero_acc()        | 0xA000     | 0   | 0      |
| 6    | (At this point, it loops forever) | 0xA000     | 0   | 0      |

We need a second instruction that tells the machine where the next instruction is located.

To remedy this, I have invented what is probably the pinnacle of this project.


<a id="orgf4a6ed5"></a>

### The jammer

What if we had a friend that would just jam another instruction into INS at step 4?

Enter modern multithreaded programming. We detach a thread whose only purpose is to jam another instruction into INS as soon as possible. This basically keeps the next instruction in memory somewhere, and could be replaced by a queue, but I think this mechanism is fun. Plus, it's definitely a write-only mechanism.

```rust
fn deferred_jam_instruction(ins: u16) {
    std::thread::spawn( move || {
        let mut cycles = 10; // Lol deadlock prevention
        while INS.load(Ordering::SeqCst) != 0 && cycles > 0 {
            cycles -= 1; 
            // Wait for INS = 0
            thread::sleep(time::Duration::from_millis(2));
        }
        INS.store(ins, Ordering::SeqCst);
    });
}
```

The op-code BBXX, loads the instruction at (XX+1) into memory, and then creates a jammer to load BB(XX + 2) into INS if INS == 0. Beautiful. If you are wondering whether this causes problems, or has race-condition issues, the answer is yes.

We can now set ACC=2 and do the next instruction by simply having INS = BB04, and having the following values in memory:

| Address | Value  | Instruction functionality                          |
|------- |------ |-------------------------------------------------- |
| 0x04    | 0xA000 | Zero ACC                                           |
| 0x05    | 0xBB06 | Set INS to value of 0x06 and start jammer for 0x07 |
| 0x06    | 0xA001 | Incr ACC                                           |
| 0x07    | 0xA001 | Incr ACC                                           |

We can chain jammers to keep the code moving

| Address | Value  | Instruction functionality                          |
|------- |------ |-------------------------------------------------- |
| 0x04    | 0xA000 | Zero ACC                                           |
| 0x05    | 0xBB06 | Set INS to value of 0x06 and start jammer for 0x07 |
| 0x06    | 0xA001 | Incr ACC                                           |
| 0x07    | 0xBB08 | Set INS to value of 0x06 and start jammer for 0x07 |
| 0x08    | 0xA001 | Incr ACC                                           |
| 0x09    | 0xAA04 | Store ACC to 0x04                                  |

Let's see this in action.

| Step | Place in code                                               | heap[0x04] | ACC | INS                |
|---- |----------------------------------------------------------- |---------- |--- |------------------ |
| 0    | Start of loop                                               | 0xA000     | any | 0xBB04             |
| 1    | Dispatch -> set_ins_and_jam() | 0xA000     | any | 0                  |
| 2    | After set_ins_and_jam()       | 0xA000     | any | 0xA000             |
| 3    | Dispatch -> zero_acc()                            | 0xA000     | any | 0xBB06             |
| 4    | After zero_acc()                                  | 0xA000     | 0   | 0xBB06             |
| 5    | Dispatch -> set_ins_and_jam() | 0xA000     | 0   | 0                  |
| 6    | After set_ins_and_jam         | 0xA000     | 0   | 0xA001             |
| 7    | (Prior to dispatch)                                         | 0xA000     | 0   | 0                  |
| 8    | Dispatch -> incr_acc()                            | 0xA000     | 0   | 0xBB08 (jammed in) |
| 9    | After incr_acc()                                  | 0xA000     | 1   | 0xBB08             |
| 10   | Dispatch -> set_ins_and_jam() | 0xA000     | 1   | 0                  |
| 11   | After set_ins_and_jam()       | 0xA000     | 1   | 0xA001             |
| 12   | Dispatch -> incr_acc()                            | 0xA000     | 1   | 0                  |
| 13   | After incr_acc()                                  | 0xA000     | 2   | 0xAA04 (jammed in) |
| 14   | Dispatch -> write_acc()                           | 0xA000     | 2   | 0                  |
| 15   | After write_acc()                                 | 2          | 2   | 0                  |
|      | (At this point, it loops forever)                           | 2          | 2   | 0                  |

Fantastic! The jammer is a very nice and not at all problematic way to do what could be done by a simple data structure. But this still doesn't solve the main problem: how is this scenario possible if all the memory values, including ACC and INS, are set to 0?


# Interacting with the uninteractive machine

So far, I have withheld one critical piece of information, which is that the program runs on a general purpose computer on which we have access to memory addresses. While this is not surprising, it does allow us to do one neat trick.

The trick is that we can write and read to values in memory. We can also interrupt the control flow. In fact, most programmers have done this at some point in their life, through a tool that exploits this same flaw; the debugger.


## Hooking up the debugger and writing a program

We are going to be using `rust-gdb` for this example, though other debuggers would certainly work.

```
rust-gdb target/debug/write-only
```

For this example, let us go through how we would run a predetermined program, then we'll go over what the program does.

We are going to set a couple breakpoints. The first one is intended to interrupt before the assignment of the value of INT to the temporary variable int. The second one is intended to intercept calls to `write_acc` so that we can change the value of ACC and thus write whatever we want into memory. The third catches the program when it is terminated, allowing us to read the value of ACC.

Let's set-up the breakpoints, run the program and continue until the first breakpoint.

```
b 23
b write_acc
b terminate
r
c
```

At the first breakpoint, we can now write our instruction to INS = 0xAB00, and continue on until we reach the second breakpoint.

```
set INS.v.value = 0xAB00
c
```

This will hit the dispatch table and end up running the function `write_acc_all`.

```rust
  fn write_acc_all(heap:&mut [u64; 256]) -> Continuation {
    for index in 0..0xFF {
        write_acc(heap, index);
    }
    Continuation::Continue
}
```

For every iteration of the loop, we have one call to `write_acc`. This is where we placed our second breakpoint.

```rust
  fn write_acc(heap:&mut [u64; 256], ins: u16) -> Continuation {
    let index: usize = (ins & 0xFF).into();
    heap[index] = ACC.load(Ordering::SeqCst);
    Continuation::Continue
}
```

If we set ACC before every call, it will load whatever value we want into memory.

```
set ACC.v.value = 0
c
```

This will continue until the next iteration of `write_acc_all`. We can proceed to mass assign our instructions and variables.

```
set ACC.v.value = 1
c
set ACC.v.value = 1000
c
set ACC.v.value = 0xBB04
c
set ACC.v.value = 0xA000
c
set ACC.v.value = 0xBB06
c
set ACC.v.value = 0xA100
c
set ACC.v.value = 0xBB08
c
set ACC.v.value = 0xA101
c
set ACC.v.value = 0xBB0A
c
set ACC.v.value = 0xAA00
c
set ACC.v.value = 0xBB0C
c
set ACC.v.value = 0xA000
c
set ACC.v.value = 0xBB0E
c
set ACC.v.value = 0xA101
c
set ACC.v.value = 0xBB10
c
set ACC.v.value = 0xA001
c
set ACC.v.value = 0xBB12
c
set ACC.v.value = 0xA001
c
set ACC.v.value = 0xBB14
c
set ACC.v.value = 0xAA01
c
set ACC.v.value = 0xC316
c
set ACC.v.value = 100
c
set ACC.v.value = 0xBA05
c
set ACC.v.value = 0xBB19
c
set ACC.v.value = 0xA000
c
set ACC.v.value = 0xBB1B
c
set ACC.v.value = 0xA100
c
set ACC.v.value = 0xFFFF
c
```

We have many more loops, but we don't have anything to write, so let's clear the second breakpoint and continue.

```
 cl write_acc
c 
```

We will hit the first breakpoint again. There is nothing left to do but launch our program.

```
  set INS.v.value = 0xBA03
cl 23
c
```

This will run until we hit termination. The value in ACC is the one we care about so

```
print ACC.v.value
```

And we are done.


### Um, what does that program do?

Well, it's supposed to sum up all the odd numbers below 100, but I found that the limit is 255 before it enters an infinite loop. It also does not output the correct value, and I haven't had time to figure out why it didn't work during the 48-hour allocated period.

I think this deserves extra points. The language is so utterly shite that even its creator couldn't produce an example that runs properly.

The version hosted on my personal github should have a version that works, at some point in the future.

Here's what the memory, and each address does

| Addr-hex | Value  | Effect                                                                                         |
|-------- |------ |---------------------------------------------------------------------------------------------- |
| 00       | 0      | Storage, summation variable "sum"                                                              |
| 01       | 1      | Increment "inc"                                                                                |
| 02       | 1000   | &#x2013;unused, kept to not mess up line numbers-                                              |
| 03       | 0xBB04 | Set INS to the value at 0x04, and jam at 0x05                                                  |
| 04       | 0xA000 | Zero ACC                                                                                       |
| 05       | 0xBB06 | &#x2026;                                                                                       |
| 06       | 0xA100 | Add "sum" to ACC                                                                               |
| 07       | 0xBB08 | &#x2026;                                                                                       |
| 08       | 0xA101 | Add "inc" to ACC                                                                               |
| 09       | 0xBB0A | &#x2026;                                                                                       |
| 0A       | 0xAA00 | Set "sum" to ACC                                                                               |
| 0B       | 0xBB0C | &#x2026;                                                                                       |
| 0C       | 0xA000 | Set ACC to 0                                                                                   |
| 0D       | 0xBB0E | &#x2026;                                                                                       |
| 0E       | 0xA101 | Add "inc" to ACC                                                                               |
| 0F       | 0xBB10 | &#x2026;                                                                                       |
| 10       | 0xA001 | Add 1 to ACC                                                                                   |
| 11       | 0xBB12 | &#x2026;                                                                                       |
| 12       | 0xA001 | Add 1 to ACC                                                                                   |
| 13       | 0xBB14 | &#x2026;                                                                                       |
| 14       | 0xAA01 | Set "inc" to ACC                                                                               |
| 15       | 0xC316 | Compare ACC to the value in 16, if false, set INS to value of 17, if true set INS to val of 18 |
| 16       | 100    | Limit for increment,                                                                           |
| 17       | 0xBA05 | Set INS to value of 05, which is 0xBB06                                                        |
| 18       | 0xBB19 | &#x2026;                                                                                       |
| 19       | 0xA000 | Zero ACC                                                                                       |
| 1A       | 0xBB1B | &#x2026;                                                                                       |
| 1B       | 0xA100 | Add "sum" to ACC                                                                               |
| 1C       | 0xFFFF | Terminate                                                                                      |

Or, if you write it in pseudocode

```python
s = 0
inc = 1
while True:
    # Doing "sum = sum + inc"
    acc = 0
    acc += s
    acc += inc
    sum = acc

    # Doing "inc = inc + 2"
    acc = 0
    acc += inc
    acc += 1
    acc += 1
    inc = acc

    if inc > 100:
        break
acc = 0
acc = s
```

This gives the right result in Python.

¯\\\_(ツ)\_/¯


# Work left to do

Other than making it work, there are also some exploits to the language, mainly thatyou can write to the heap directly. It's too convenient, making the language half-usable. As such, there needs to be some indirection nonsense to make this more inconvenient than the method I presented before. I have not yet conceptualized what I intend for this.
