---
title: Process Synchronisation
permalink: /os_labs/lab4_partb
key: labs-tinyos
license: false
layout: article
nav_key: os_labs
sidebar:
  nav: os_labs
license: false
aside:
  toc: true
show_edit_on_github: false
show_date: false
---

Finally, you need to **coordinate** the operation of the user-mode processes so that click messages only appear <span style="color:red; font-weight: bold;">after</span> the prompt has been output but before you have started typing in a sentence to be translated. This can be done using semaphores.

## Part B: Synchronize mouse reporting with other I/O using Semaphores

In other words, once you start typing in a sentence, click messages should be **delayed** until after the next prompt. If the user clicks _multiple_ times after they have started typing, only a SINGLE click message needs to be displayed (describing either the **first** or **last** click, your **choice**).

You may declare a mouse semaphore in P3, and immediately `Wait` (attempt to decrease) for the Semaphore:

```nasm
MouseSemaphore: semaphore(0)	| Semaphore for mouse, initialised from zero

P3Start:

	Wait(MouseSemaphore) 	| proceed only when prompt has shown
    | ... implement printing of click coordinates here

    Signal(Prompt) | signal the prompt so it will print another prompt
    | ... continue implementation

```

### Non-blocking SVC

You also need two more supervisor calls to check for any keyboard press and check for any mouse click that is **non blocking** because we need to know whether we have typed something (and block the mouse click printout in P3):

```nasm
.macro CheckMouse() SVC(9) 	| Part D: TO CHECK MOUSE CLICK, NON BLOCKING
.macro CheckKeyboard() SVC(10) 	| Part D: TO CHECK KEYBOARD CLICK, NON BLOCKING
```

Update the corresponding `SVC_tbl` to support these two:

```nasm
SVCTbl:	UUO(HaltH)		| SVC(0): User-mode HALT instruction
	UUO(WrMsgH)		| SVC(1): Write message
    ...
	UUO(CheckMouseH)| SVC(9) : CheckMouse()
	UUO(CheckKeyH)	| SVC(10) : CheckKeyboard()
```

The implementation of the two service handlers above is suggested to be as follows:

```nasm
||| LAB 4 PART B: add new handler to check keyboard state, but doesn't clear it and doesn't block the calling process
CheckKeyH:
	LD(Key_State, r0)
	ST(r0,UserMState)		| return it in R0.
	BR(I_Rtn)			| and return to user.

||| LAB 4 PART B: add new handler to check mouse state, but doesn't clear it and doesn't block the calling process
CheckMouseH:
	LD(Mouse_State, r0) 	| put the content of Mouse_State to R0
	ST(r0,UserMState)		| return it in R0 of the user state since UserMState points to the R0 of the user reg value
	BR(I_Rtn)			| and return to user
```

And then, somewhere in P0 **after** the prompt is printed out, you should check whether there exist mouse click **OR** keyboard click, and `signal` (increase) the semaphore accordingly:

```nasm
P0Read:	Wait(Prompt)		| Wait until P1 has caught up...
	WrMsg()			| First a newline character, then
	.text "\n"
	LD(Count3, r0)		| print out the quantum count
	HexPrt()		|  as part of the count, then
	 WrMsg()		|  the remainder.
	.text "> "
	LD(P0LinP, r3)		| ...then read a line into buffer...

||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
||||| LAB 4 PART B: TO SYNCHRONISE, busy wait
beginCheckMouse: CheckMouse()
    BEQ(R0, beginCheckKeyboard) | go and check for keyboard press if there's no mouse click
                                | If there's no mouse click, P3 is stuck at GetMouse() anyway
    Signal(MouseSemaphore)		| if there mouse click, give signal for P3 to continue
    Yield()         | stop execution
    BR(P0Read)		| and restart process

beginCheckKeyboard: CheckKeyboard()
    BEQ(R0, beginCheckMouse)
||||| END OF Part D
||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||

P0RdCh: GetKey()		| read next character,
	WrCh()			| echo back to user
    ...
```

### `Task 2`{: .info}

At the end, you should be able to report mouse clicks: one click printed per click, doesn't matter if you spammed. Notice the number within the prompt must also increase (proving that `P2` is scheduled properly).

When youre in the middle of typing something, e.g: `Hello` in the example, any click should **not** cause you to print any mouse coordinates **until after** the user **entered** the message.

<img src="{{ site.baseurl }}/assets/contentimage/lab6/9.gif"  class="center_seventy"/>

`CHECKOFF`{:.info}
Demonstrate the above result of <span style="color:red; font-weight: bold;">delayed</span> printing the mouse coordinates (when the mouse clicks at the terminal area and user has started typing) to your instructor / TA in class. Only one mouse click should be reported.
{:.info}

## Summary

Notice that `P0` doesn't have to confirm until `P3` has finished one round of execution (printing of x, y coordinate) _before_ restarting to `BR(P0Read)` because we **know** that the round robin scheduler will **surely** execute P3 for a round once P0 calls `Yield()`.

Thanks to the scheduler's round robin policy and long enough quanta dedicated for each process, there won't be the undesirable condition whereby P0 `Yield()` immediately returns execution to P0 again, **before** P3 resumes and then `Signal` the `MouseSemaphore` the **second** time (because it hasn't been cleared by P3 that hasn't progressed!).

Without the round robin policy, `MouseSemaphore` value might accidentally be increased to 2 and we might have a future `Click` message printed out at the same time **while** typing some messages at the console, violating the condition required for this Part 4. If we want to fix this (e.g: assume there's priority scheduling policy instead fo round robin policy), we might have to check that a new mouse click is _actually made_ in `CheckMouseH` by storing the _previous_ history of mouse click at all times.
