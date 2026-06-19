# Minesweeper Reverse Engineering Write-up 💣

## Part A
---
To solve this Part, I realized the best approach was to first figure out how the game stores its data behind the scenes, and only then try to write the actual patch. Because of that, I'll start with questions 2 and 3 to build the foundation, and then explain the patch itself (Question 1).

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

<img width="291" height="11" alt="image" src="https://github.com/user-attachments/assets/e83aff8e-433b-4f07-9332-c1b651dc8a49" />


This was the breakthrough. In Win32 programming, lpfnWndProc points to the Window Procedure (WndProc)—the core function responsible for intercepting and handling every single event (clicks, keyboard inputs, resizing) sent to the window. This told me that `sub_1001BC9` is the main engine running the game logic.

### Tracking the Mouse Click to the Board Address

I entered `sub_1001BC9` and followed its internal jump table to follow the left mouse click event (`WM_LBUTTONDOWN`, which is `0x201`).
Here, I analyzed exactly how the program calculates which cell was clicked:
* The raw mouse pixel coordinates are passed via `lParam`. The X-coordinate is stored in the low-order word, and the Y-coordinate is in the high-order word. 
* The code extracts the Y-coordinate and performs a bitwise arithmetic shift right: `sar eax, 4`. Shifting a binary number right by 4 bits is mathematically equivalent to dividing it by $2^4$ (which is 16), discarding the remainder.
* Since every single square on the grid is exactly 16x16 pixels, this division converts the raw screen pixels into a simple row and column index (for example: Row 3, Column 5).

<img width="178" height="62" alt="image" src="https://github.com/user-attachments/assets/ee22e68c-48f9-4120-bcbb-c403c9d4c782" />

Right after calculating the row index, the game invokes sub_10031D4 and passes this row index to access the actual grid data structure.

When the game needs to redraw or update a specific row on the screen (triggered during window paint and update events inside sub_1001BC9), it uses a fast bitwise calculation in the loc_1001E59 block to find the correct memory offset:
<img width="148" height="32" alt="image" src="https://github.com/user-attachments/assets/6c26de43-0ede-4c91-be88-8d42c4153fda" />

* The lea line multiplies the row index inside eax by 3.
* The shl line shifts that result left by 2 bits, which multiplies it by 4.

Together, 3 * 4 = 12. This calculation means that each row structure takes up exactly 12 bytes in this internal state lookup table. The program then uses this 12-byte row offset to pull the target row pointer from the address array `dword_1005010`.

By analyzing how these internal drawing updates and state lookups map out in assembly, I discovered the final, static base memory address where the structural board data actually begins: `0x01005340`.

### Dynamic Analysis & Verification
To prove this was the real board, I fired up a dynamic debugger, ran the game, and paused it. I navigated to the calculated base address 0x01005340 in the Memory Dump window.

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
