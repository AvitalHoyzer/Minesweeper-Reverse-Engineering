# Minesweeper Reverse Engineering Write-up 💣

To solve this challenge, I realized the best approach was to first figure out how the game stores its data behind the scenes, and only then try to write the actual patch. Because of that, I'll start with questions 2 and 3 to build the foundation, and then explain the patch itself (Question 1).

## 2. Where is the game board located in memory? (And how I found it)

Instead of just guessing memory addresses, I started by looking for my most basic interaction with the game: a mouse click.

### Entering the Main Function
When I first loaded `winmine.exe` into IDA, the execution started at the standard `start` routine (the C Runtime initialization code). I scrolled past the boilerplate code until I found the jump to the real application entry point: a call to **`sub_10021F0`**. 

When I entered `sub_10021F0`, I immediately recognized it as the classic Win32 `WinMain` function. Looking at it globally, I saw it doing three main things:
1. Allocating and initializing a `WNDCLASSW` structure to define the window properties.
2. Registering the window class with the OS using `RegisterClassW`.
3. Creating the actual window using `CreateWindowExW` and starting the standard Windows message loop (`GetMessageW` / `DispatchMessageW`).

### Finding the WndProc (The Brains of the Window)
Inside `WinMain`, while the window properties were being configured, I looked closely at where the window procedure callback was being assigned:
```assembly
mov [ebp+WndClass.lpfnWndProc], offset sub_1001BC9
```
This was the breakthrough. In Win32 programming, lpfnWndProc points to the Window Procedure (WndProc)—the core function responsible for intercepting and handling every single event (clicks, keyboard inputs, resizing) sent to the window. This told me that sub_1001BC9 is the main engine running the game logic.

### Tracking the Mouse Click to the Board Address

I entered `sub_1001BC9` and followed its internal jump table to follow the left mouse click event (WM_LBUTTONDOWN, which is 0x201).
Here, I analyzed exactly how the program calculates which cell was clicked:
*The raw mouse pixel coordinates are passed via `lParam`. The X-coordinate is stored in the low-order word, and the Y-coordinate is in the high-order word.
*The code extracts the Y-coordinate and performs a bitwise arithmetic shift right: `sar eax, 4`. Shifting a binary number right by 4 bits is mathematically equivalent to dividing it by $2^4$ (which is 16), discarding the remainder.Since every square on the Minesweeper board is exactly 16x16 pixels, this division converts the raw pixel position on the screen into a logical row and column index (e.g., Row 3, Column 5).To map this 2D coordinate (Row, Column) into a 1D memory array, the game unrolls the loop using a fixed row width of 32 bytes. I found the following key instruction:
shl ecx, 5
Shifting the row index left by 5 bits is equivalent to multiplying it by $2^5$ (which is 32). The final memory address of a clicked cell is calculated using the following formula:$$\text{Cell Address} = \text{Base Address} + (\text{Row} \times 32) + \text{Column}$$By looking at where this calculated address pointed in the assembly, I discovered the fixed, static base memory address of the board: 0x01005340.
Step 4: Dynamic Debugging Verification
To prove this was the real board, I fired up a dynamic debugger (x64dbg), ran the game, and paused it. I navigated to the calculated base address 0x01005340 in the Memory Dump window.

As seen in the screenshot below, a clear grid structure is visible, beautifully outlined by 10h values which act as the invisible "walls" or borders of the board, confirming the calculation was 100% correct.

## 3. What values can a cell on the board hold?

Once I found the board, I needed to understand what the hex numbers actually meant. Reading the assembly in IDA revealed that the game uses bitwise logic and masks to determine a cell's state:

* **Is there a mine? (Bit 7 - The MSB):** In the functions that generate mines or check if you lost, I saw the instruction `test al, 80h`. The hex value `80h` is `10000000` in binary. This is the "Mine Flag". If this specific bit is turned on, the cell contains a mine.
* **Is the cell opened? (Bit 6):** In the drawing mechanism, I found the instruction `test al, 40h` (`01000000` in binary). This bit tracks whether the cell is still hidden or has already been clicked and revealed to the player.
* **The visual appearance (Bits 0-4):** To decide which "sprite" (image) to draw on the screen, the game masks out the top bits using `and al, 1Fh` (isolating the bottom 5 bits). The values left over tell us what's drawn:
  * `0Fh`: A standard hidden/closed square.
  * `0Eh`: A square with a flag on it.
  * `0Ah`: A square with a question mark.
  * `00h` to `08h`: An opened square showing a number (0 to 8).

**Putting it all together:**
* When a game starts, the board is filled with `0Fh` (empty hidden squares).
* When the game plants a mine, it takes that `0Fh` and does a bitwise `OR` with `80h` (the mine bit), resulting in **`8Fh`**.
* So, `8Fh` means a hidden mine. If we want a mine that also visually shows a flag, we need to combine the mine bit (`80h`) with the flag sprite (`0Eh`), which gives us **`8Eh`**.

## 1. How did we do it? (The Patch)

The goal of the assignment was to make the game launch with all the mines already flagged, without the timer even starting.

Because I already understood how the memory works, the patch ended up being incredibly simple and required modifying just a few bytes in the `.exe` file.

**My research and patching process:**

1. I searched IDA for the function responsible for randomly generating and placing the mines on the board. I found it at `sub_100367A`. Interestingly, I noticed this function is actually called as soon as the game loads (not just after the first click).
2. Inside the mine-generation loop, I found the exact instruction that plants the mine into memory (at address `010036FA`):

   ```assembly
   or byte ptr [eax], 80h
