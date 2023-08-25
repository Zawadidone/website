---
name: Reverse Engineering for Beginners
date: 2020-05-01
header: 
  teaser: /assets/img/EWjMtkyXQAYLh4c.png
---

Another reverse engineering challenge :)

### Password

Opening the binary Ghidra shows the following pseudo-code of the function named `entry`.

```c
int entry(void)

{
  [...]
  unaff_ESI = FUN_00401090(); // this is the main function
    [...]
}
```

The function `entry` executes the function named `FUN_00401090` renamed to `main`. The function asks for a string using  `scanf` and checks the input against the string "cr4ckm3" with using `strcmp`. If `strncmp` returns `0` the password is correct.

![image-20200313231544489]({{site.url}}/assets/img/image-20200313231544489.png)

```powershell
PS C:\Users\re\Documents> .\Exer0_Password.exe
Enter password: cr4ckm3
Correct!
```

### Good_Luck

The function `entry` calls `FUN_00401040` renamed to `main`:

```c
int entry(void)
{
  [...]
    unaff_ESI = FUN_00401040(*piVar8,iVar5);
  [...]
}
```

![image-20200313234548106]({{site.url}}/assets/img/image-20200313234548106.png)

`atoi` converts `argv[1]` to an integer. If `argv[1]` contains a string and not an integer the function returns `0`.

```powershell
PS C:\Users\re\Documents> .\Exer1_GoodLuck.exe AAAA
PS C:\Users\re\Documents>
```

After checking if `argv[1]` contains an integer the function checks if the value of `argv[1]` multiply `5` equals `6170`. If true the function prints "Verry Correct!".

```powershell
PS C:\Users\re\Documents> python
>>> 6170 / 5
1234
PS C:\Users\re\Documents> .\Exer1_GoodLuck.exe 1234
Very correct!
```

### Julia

The first part of the function `main` checks for a second argument, saves the length in the variable `input_length` and copies the argument to the heap with the use of `malloc` and `strcpy`.

![image-20200314182504546]({{site.url}}/assets/img/image-20200314182504546.png)

After copying the argument to the heap, every character is "checked" by the function `input_check` in a while loop. If the result of the while loop saved in the array `input_heap` minus the `input_length` equals to the string `VIMwXliFiwx` the function prints `Brava!!!`.

![image-20200314182349534]({{site.url}}/assets/img/image-20200314182349534.png)

The goal of the challenge is to determine what the function `input_check` does and what input we have to enter to get the string `VIMwXliFiwx`. The function `input_check` performs 3 actions on the string `input` explained in the pseudo code below. The character values used are [ASCII characters](https://www.asciitable.com/).

![image-20200314182207193]({{site.url}}/assets/img/image-20200314182207193.png)

To reverse the check of the input of the string `VIMwXliFiwx` I recreated the function `input_check` in Python.

```python
from string import printable

ascii = {}

for s in printable:
    input = ord(s)

    if 96 < input and input < 123:
        input += 4

    input_var = input
    if 64 < input and input < 91:
        input += 4

    input = chr((input_var >> 7) + input)

    ascii[input] = s

flag = "VIMwXliFiwx"
result = ""
for f in flag:
    result += ascii[f]

print(result)
```

```bash
re@DESKTOP-S7VKEO8:/mnt/c/Users/re/Documents$ python flag.py
REIsTheBest
```

```powershell
PS C:\Users\re\Documents> .\Exer2_Juila.exe REIsTheBest
Brava!!!
```

After finishing the challenge I figured out I also could have used an [online ceasar cypher website](https://cryptii.com/pipes/caesar-cipher), but I like Python and recreating the function in a different programming language.

![image-20200315234730278]({{site.url}}/assets/img/image-20200315234730278.png)

### Minesweeper

The goal is to make Minesweeper print the flags on the positions where there are mines when starting up the game.

Opening the binary in Ghidra and looking for references to the function `rand` shows an incoming reference from the function `FUN_01003940` renamed to `rand_caller`:

![image-20200424173841615]({{site.url}}/assets/img/image-20200411214500608.png)

The function `rand_caller` gets called two times from the function `FUN_0100367a`. 

![image-20200424175145694]({{site.url}}/assets/img/image-20200424175145694.png)

The values passed to the function `rand_caller` are at the addresses `01005334` and `01005338`. After that, the mines are set with the instruction `OR byte ptr [y-as],0x80`.

![image-20200426201544688]({{site.url}}/assets/img/image-20200426201544688.png)

Let's debug the binary and watch the values at the address `0x01005330 `. What stood out is the value `0xA`. This is the number of mines in the game. 

![image-20200426144330291]({{site.url}}/assets/img/image-20200426144330291.png)

The memory contents shows the mine field with the following values.

* 0x01005330 = mines = 0xA
* 0x01005334 = x-as = 0x9
* 0x01005338 = y-as = 0x9
* 0x10 = border
* 0x8F = mine
* 0x0F = empty

To cheat the game we can print the contents of memory and show where the mines are or show the mines at the start of the game with a flag. Let's try modifying the binary to set the value of `0x8E` (flag) instead of `0x8f` (mine).`OR byte ptr [y-as],0x80` changed to `mov byte ptr ds:[eax], 0x8E`

![image-20200426202349775]({{site.url}}/assets/img/image-20200426202349775.png)

Yes, it shows the mines :)

![image-20200426202517748]({{site.url}}/assets/img/image-20200426202517748.png)

While playing it marks the mines with flags to make it easier to cheat.

![]({{site.url}}/assets/img/EWjMtkyXQAYLh4c.png)
