# DISTINCT

We first start off by looking at the source code.

```
typedef void (*hh)(void);

unsigned long long nums[SZ];
hh handler;
```

We see that handler is a function pointer; i.e., it points to a memory address which is supposed to be the start of a function.
We see that 16 unsigned long long ints (8 bytes) of nums (SZ is defined as 0x10 which is 16) is first pushed onto the stack, then our function pointer handler is pushed onto the stack. Remember that the stack is first in last out, so addresses are pushed to the bottom of the stack first.
![This is what our stack should look like in this part.](https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/stack.png)
