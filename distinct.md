# DISTINCT

We first start off by looking at the source code.

```
typedef void (*hh)(void);

unsigned long long nums[SZ];
hh handler;
```

We see that handler is a function pointer; i.e., it points to a memory address which is supposed to be the start of a function.
We see that 16 unsigned long long ints (8 bytes) of nums (SZ is defined as 0x10 which is 16) is first pushed onto the stack, then our function pointer handler is pushed onto the stack. Remember that the stack is first in last out, so addresses are pushed to the bottom of the stack first.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/stack.png" width="400"/>

This is what our stack should look like at this point of time.

```
handler = &unique;

```
In our check() function, handler first points to a unique() function. 

```
for (int i = 0; i < SZ-1; i++) {
  if (nums[i] == nums[i+1]) {
    handler = &repeated;
  }
}
```

After entering our 16 numbers, a check is done if any of the numbers are repeated, and if it is we point at a repeated() function instead.

```
void win() {
    system("/bin/sh");
}
```

This is the function we want handler to point to, the question right now is how we would get handler to point to. The biggest hint we get is actually in the challenge description ("Except the sort looks off ..."), so let's look at the sort.

```
void sort(unsigned long long* arr, unsigned long long len) {
    unsigned long tmp = 0;
    for(int i = 0; i <= len; i++) {
        for(int j = i; j <= len; j++) {
            if (arr[i] < arr[j]) continue;
            tmp = arr[i];
            arr[i] = arr[j];
            arr[j] = tmp;
        }
    }
}
```

At first glance, this looks like a fairly standard bubble sort, so lets look at how the function is called.
After we enter our numbers, the program does this:

```
sort(nums, SZ);
```

Hang on, SZ is defined as 16, and i and j stop when they are equal to 16, so we are actually accessing nums[16], or one number on top of nums[15], our last inputted digit in the stack, which, as shown in the stack diagram earlier, is equal to handler. Essentially, handler() is now included in our sort, and handler() is the last number that is in the function scope. Hence, handler() is going to end up either as the largest number in our array, or the initial value of handler(), whichever is larger, at the end of our sort.

But what is handler() in the first place? Handler() points to a memory address, which in a 64 bit system, is an 8 byte address. Unsigned long long integers are also 8 bytes, we can just convert the 8 byte hex address of handler() to a decimal to see what it is. We see that handler points to unique() first, so let's run "p unique" in gdb to see what memory address unique() points to.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/p_unique.png"/>

Next, let's run "p win" to see where we want to go.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/p_win.png"/>

Great, the address of win() is larger than the address of unique, so if we enter the address of win() as a decimal number (and we don't enter even larger numbers), we should be able to change the address that handler() is pointing to, to win, once the sort function completes.

Let's try it in gdb. We can convert the hex address of win to decimal conveniently just by entering it into python2.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/python_hexconvert.png"/>

In gdb, we're just going to enter 15 numbers that don't matter, and then the value of win (93824992236948) that we got, then enter 'N' to continuing, so that the check() function enters handler(), which should point to win().

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/gdb_execution.png", width="400"/>
