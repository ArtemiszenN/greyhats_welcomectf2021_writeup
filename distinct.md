# DISTINCT

We first start off by looking at the source code.

```
typedef void (*hh)(void);

unsigned long long nums[SZ];
hh handler;
```

We see that handler is a function pointer; i.e., it points to a memory address which is supposed to be the start of a function, and that 16 unsigned long long ints (8 bytes) of nums (SZ is defined as 0x10 which is 16) is first pushed onto program data, then our function pointer handler is pushed onto program data. We can verify this in gdb by disassembling check, then looking at the memory at where nums is.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/disassemble_check.png"/>
<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/disassemble_check2.png"/>

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

Hang on, SZ is defined as 16, and i and j stop when they are equal to 16, so we are actually accessing nums[16], or one number on top of nums[15], our last inputted digit in the stack, which, as shown in the stack diagram earlier, is equal to handler(). Essentially, handler() is now included in our sort, and handler() is the last number that is in the function scope. Hence, handler() is going to end up either as the largest number in our array, or the initial value of handler(), whichever is larger, at the end of our sort.

But what is handler() in the first place? Handler() points to a memory address, which in a 64 bit system, is an 8 byte address. Unsigned long long integers are also 8 bytes, we can just convert the 8 byte hex address of handler() to a decimal to see what it is. We see that handler points to unique() first, so let's run "p unique" in gdb to see what memory address unique() points to.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/p_unique.png"/>

Next, let's run "p win" to see where we want to go.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/p_win.png"/>

Great, the address of win() is larger than the address of unique, so if we enter the address of win() as a decimal number (and we don't enter even larger numbers), we should be able to change the address that handler() is pointing to, to win(), once the sort function completes.

Let's try it in gdb. We can convert the hex address of win() to decimal conveniently just by entering it into python2.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/python_hexconvert.png"/>

In gdb, we're just going to enter 15 numbers that don't matter, and then the value of win (93824992236948) that we got, then enter 'N' to continue, so that the check() function enters handler(), which should point to win().

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/gdb_execution.png" width="400"/>

Cool, so we now have code redirection in gdb. Let's try it on our actual machine.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/local_fail.png" width="400"/>

It didn't work. Sometimes, we would also get a segmentation fault and our program would crash. Running "file distinct" (I renamed the file from distinct.o to distinct), we can easily see why that is the case.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/file_distinct.png"/>

We can see that distinct is referred to as an "ELF 64-bit LSB shared object", instead of as  "ELF 64-bit LSB executable". That means that our program has PIE (Position Independent Executable) enabled. In other words, when our program is loaded into virtual memory, the addresses which it is given is randomized each time. This is disabled in gdb, which is why our exploit was able to work. 

The good news is that the relative addresses of our functions are the same, and since we were previously able to get the addresses of our win() and unique() functions in gdb,  we can just subtract the two to get the offset that we need. We can do this pretty easily in python2.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/python_hexsubtract.png"/>

We can also find the offsets by using objdump -t distinct.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/objdump.png"/>

The first number in that output represents the offset of our function from the text section in our program. In gdb, we can find where that is with "info proc map".

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/infoprocmap.png"/>

We can see our text section starts at 0x555555554000. Adding the offset of win, 0000000000001594, to that, we get 0x555555555594, which confirms our objdump output. Hence, we can either take 0x555555555594 - 0x55555555537d or 0000000000001594 - 000000000000137d to get our offset of win() from unique(), which both give us 535. Of course, we also have to remember not to enter the same number, or handler() would point to repeated(). Remember that the check for repeated numbers is done after sorting, so our inserted address would get replaced by the address to repeated() if we enter repeated numbers.

Next problem: we need to find the address to unique() so that we can add the output. This is actually a rather easy problem to solve, because the program prints all the sorted numbers before asking us if we want to go again. If we can get the address of unique() into our array, we can get the address printed in decimal. Add 535 to that, and we get our address of win() in decimal. We don't know what the address of unique() is though, so let's just enter the largest possible 8 byte number and check if we get something. We're going to do this on the challenge server since we should be rather close to the solution.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/distinct_cropped.png"/>

Cool, the number printed is clearly not the number we inputted, so it should be the address of unique() in decimal. We just have to add 535 to that in our next loop and stop the program there to get a shell. Remember we have a 60 second timeout window, so we either have to be fast or automate this in python.

<img src="https://github.com/ArtemiszenN/greyhats_welcomectf2021_writeup/blob/main/img/distinct_c.png"/>

There we go.
