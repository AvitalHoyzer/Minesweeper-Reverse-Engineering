# Minesweeper Reverse Engineering Write-up 💣

## Part A
---
To solve this Part, I realized the best approach was to first figure out how the game stores its data behind the scenes, and only then try to write the actual patch. Because of that, I'll start with questions 2 and 3 to build the foundation, and then explain the patch itself (Question 1).

## 2. Where is the game board located in memory? (And how I found it)

Instead of just guessing memory addresses, I started by looking for my most basic interaction with the game: a mouse click.

### Entering the Main Function
When I first loaded winmine.exe into IDA, the execution started at the standard start routine (the C Runtime initialization code). I scrolled past the boilerplate code until I found the jump to the real application entry point: a call to **sub_10021F0**. 

When I entered sub_10021F0, I immediately recognized it as the classic Win32 WinMain function. Looking at it globally, I saw it doing three main things:
1. Allocating and initializing a WNDCLASSW structure to define the window properties.
2. Registering the window class with the OS using RegisterClassW.
3. Creating the actual window using CreateWindowExW and starting the standard Windows message loop (GetMessageW / DispatchMessageW).

### Finding the WndProc (The Brains of the Window)

Inside WinMain, while the window properties were being configured, I looked closely at where the window procedure callback was being assigned:

<img width="291" height="11" alt="image" src="https://github.com/user-attachments/assets/e83aff8e-433b-4f07-9332-c1b651dc8a49" />


This was the breakthrough. In Win32 programming, lpfnWndProc points to the Window Procedure (WndProc) - the core function responsible for intercepting and handling every single event (clicks, keyboard inputs, resizing) sent to the window. This told me that `sub_1001BC9` is the main engine running the game logic.

### Tracking the Mouse Click to the Board Address

I entered sub_1001BC9 and followed its internal jump table to follow the left mouse click event (WM_LBUTTONDOWN, which is 0x201).
Here, I analyzed exactly how the program calculates which cell was clicked:
* The raw mouse pixel coordinates are passed via `lParam`. The X-coordinate is stored in the low-order word, and the Y-coordinate is in the high-order word. 
* The code extracts the Y-coordinate and performs a bitwise arithmetic shift right: `sar eax, 4`. Shifting a binary number right by 4 bits is mathematically equivalent to dividing it by $2^4$ (which is 16), discarding the remainder.
* Since every single square on the grid is exactly 16x16 pixels, this division converts the raw screen pixels into a simple row and column index (for example: Row 3, Column 5).

<img width="178" height="62" alt="image" src="https://github.com/user-attachments/assets/ee22e68c-48f9-4120-bcbb-c403c9d4c782" />

Right after calculating the indexes, the game invokes sub_10031D4 to process the coordinate update. Inside the grid lookup block (at address 0x0100213A), the program converts the 2D Row and Column coordinates into a flat 1D array index:

<img width="188" height="39" alt="image" src="https://github.com/user-attachments/assets/475475c2-483a-4b24-ba95-9a8a9bb98ba1" />

 * The `shl eax, 5` line shifts the row index inside eax left by 5 bits. Shifting left by 5 bits is mathematically equivalent to multiplying by 32.
 * This explicitly tells us that each row layout on the board allocation structure spans exactly 32 bytes of width in memory.

The program then grabs the targeted cell state by taking the base address pointer byte_1005340, adding the 32-byte row stride offset (eax), and adding the column index offset (ecx). This hardcoded reference proves that the board data structurally begins at memory address 0x01005340.


### Dynamic Analysis & Verification

To prove this was the real board, I fired up a dynamic debugger, ran the game, and paused it. I placed a Breakpoint on the function return instruction at address 0x0100374E (the end of the mine deployment loop).

I then navigated directly to the base address 0x01005340 in Hex View. 

As seen in the screenshot below, a clear grid structure is visible, beautifully outlined by 10h values which act as the invisible "walls" or borders of the board, confirming the structural layout calculation was 100% correct.

<img width="343" height="262" alt="image" src="https://github.com/user-attachments/assets/05284c4b-7d82-45d5-9eeb-f0c28a380139" />



## 3. What values can a cell on the board hold?

Once I found the board, I needed to understand what the hex numbers actually meant. Reading the assembly in IDA revealed that the game uses bitwise logic and masks to determine a cell's state inside the drawing and state-updating functions:

### Checking Management Bits (Bit 6 & Bit 7)
Inside the cell-updating function at address **0x01002FA4** (sub_1002F80), we can see how the game evaluates the status flags of each square:

<img width="341" height="206" alt="image" src="https://github.com/user-attachments/assets/6e5088b6-99e0-498f-9b5a-5b0de33fb55d" />

* **Is the cell opened? (Bit 6):** The instruction `test al, 40h` (01000000 in binary) checks if bit 6 is set. If this bit is on, the cell has been clicked and revealed; if it is 0, the cell remains hidden.

* **Is there a mine? (Bit 7 - The MSB):** Further down in the same block, the program executes a sign-flag check (`test al, al` followed by jns). This natively checks if the highest bit (Bit 7, or 80h / 10000000 in binary) is active, which serves as the "Mine Flag".

### Isolating the Visual Appearance (Bits 0-4)
To decide which "sprite" (image asset) to draw on the screen, the game strips away the management bits to find the visual state. This occurs inside the drawing routine at address 0x0100266F (sub_1002646):

<img width="183" height="47" alt="image" src="https://github.com/user-attachments/assets/46476101-18af-4bf5-a429-7d85ae4ac09d" />

The bitwise mask and edx, 1Fh isolates the lowest 5 bits (00011111 in binary). The resulting value acts as an index to pull the correct sprite pointer from the graphic array `hdcSrc`:

  * **0Fh**: A standard hidden/closed square.
  * **0Eh**: A square with a flag on it.
  * **0Ah**: A square with a question mark.
  * **00h** to **08h**: An opened square showing a number (0 to 8).
 
**Putting it all together:**
* When a game starts, the board is filled with 0Fh (empty hidden squares).
* When the game plants a mine, it takes that 0Fh and does a bitwise OR with 80h (the mine bit), resulting in **8Fh** (a hidden mine).
* So if we want a mine that also visually shows a flag, we need to combine the mine bit (80h) with the flag sprite (0Eh), which gives us **8Eh**.

## 1. How did I do it? (The Patch)

The goal of the assignment was to make the game launch with all the mines already flagged, without the timer even starting.

Because I already understood how the memory and bitmasks work, the patch ended up being incredibly simple and required modifying just a few bytes in the .exe file.

**My research and patching process:**

**Locating the Mine Placement:**

1. I searched for the function responsible for randomly generating and placing the mines on the board. I found it at `sub_100367A`. Interestingly, I noticed this function is actually called as soon as the game loads (not just after the first click).
2. Inside the mine-generation loop, I found the exact instruction that plants the mine into memory (at address `010036FA`):

<img width="218" height="92" alt="image" src="https://github.com/user-attachments/assets/9d97a8a3-4c78-4a6f-aa16-63a8036ee4b2" />

