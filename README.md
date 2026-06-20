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


#### A Note on Stride Variation

During the analysis, I observed that the game uses two different 'strides' for memory navigation: a 12-byte stride within the graphical rendering routines (to look up sprite pointers in the hdcSrc array) and a 32-byte stride within the logic-heavy routines (to access the raw mine data in the board array). While these methods serve different purposes—one for the graphical interface and one for the core game logic—they both ultimately synchronize to reference the same static board base address at 0x01005340.

Graphical lookup table navigation using a 12-byte stride optimization(at address 0x01001E6A):

<img width="194" height="217" alt="image" src="https://github.com/user-attachments/assets/35089873-2d35-41bd-a581-b2e655ae9940" />

The graphical routine uses `lea eax, [eax+eax*2]` (multiplying by 3) followed by `shl eax, 2` (multiplying by 4). This results in a total multiplier of 12, creating a 12-byte stride used specifically for navigating the GUI pointer table.

## 3. What values can a cell on the board hold? (Dynamic Analysis)

I opened my debugger, ran the game, and focused on the board's memory address (0x01005340).

### What I Observed
I played around with the game and watched how the numbers in memory changed:

**The Grid Layout:** I saw that the board is bordered by 10h, and inside, there was either **8Fh** or **0Fh**. I understood that one of them is a hidden mine and the other is empty.

<img width="557" height="488" alt="image" src="https://github.com/user-attachments/assets/469cecfe-f154-4ce7-8a0f-197cbd2c2ebc" />

**Flagging:** I placed a few flags on the board:

<img width="570" height="503" alt="image" src="https://github.com/user-attachments/assets/413dae7c-b4b9-47d7-9231-9be3f676e337" />

After pause:

<img width="438" height="230" alt="image" src="https://github.com/user-attachments/assets/3706d632-f25a-4d97-93b8-938226f13e7d" />

I saw that what was 8Fh turned into **8Eh**, and what was 0Fh turned into **0Eh**.

**Question Marks:** I kept playing and tried question marks:

<img width="407" height="496" alt="image" src="https://github.com/user-attachments/assets/59c13046-728c-4406-9ad0-db2bb626e0cd" />

After pause:

<img width="343" height="239" alt="image" src="https://github.com/user-attachments/assets/0fbff84c-7e6a-4920-b60a-8bd2f6546db3" />

the value turned into **0Dh**.

**Opening Squares:** I tried to click on what was 0Fh and I understood they were empty:

<img width="504" height="496" alt="image" src="https://github.com/user-attachments/assets/2c834104-a594-4c8d-95fd-c9a6c49574fd" />

After pause:

<img width="379" height="238" alt="image" src="https://github.com/user-attachments/assets/27b80a43-8922-447f-906f-d3f76e9d3cb5" />

When I paused the game, I saw that squares with a number on them became **41h**, and completely empty squares became **40h**.

**Mines:** I understood that the ones that were 8Fh and then 8Eh were likely the mines.
When I clicked on what was 8Fh, I saw that it was indeed the mine!

<img width="527" height="506" alt="image" src="https://github.com/user-attachments/assets/635c210e-3948-4aef-bc4b-808f23b67d42" />

After pause:

<img width="394" height="235" alt="image" src="https://github.com/user-attachments/assets/99c527ee-3751-458d-8840-806b4db8413a" />

**Game Over:** After pausing, I saw that a mine I clicked turned into CCh. Also, when I put a flag on a square that wasn't a mine and the game ended, it turned into **0Bh**.

By doing this, I understood the meaning of all the values a cell on the board can receive.

### Explaining the Bitwise Logic

To satisfy the requirements, here is the connection between the hex values in memory and the display on the screen, based on the bitwise logic I found in the code.

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

<img width="218" height="95" alt="image" src="https://github.com/user-attachments/assets/8d807644-f80d-49a0-b510-4eef1655e7af" />

### The Patch Idea:
* **Original Behavior:** The original instruction used `or byte ptr [eax], 80h` which combined the 0Fh (blank hidden state) with the mine flag (80h) to yield 8Fh (a hidden, unflagged mine).
* **The Mod:** Instead of letting the game place a hidden mine, I wanted to force it to write the value for a flagged mine (8Eh) directly into memory. That way, when the window's paint function scans the board to draw it, it will immediately see the flags and render them from the very first frame.

### Execution & Hex Editing:
I replaced the logical OR instruction with a direct MOV instruction to hardcode our desired value.
* Original Assembly: or byte ptr [eax], 80h -> Hex Bytes: 80 08 80
* Patched Assembly: mov byte ptr [eax], 8Eh -> Hex Bytes: C6 00 8E

Since both instructions take up the exact same amount of bytes (3 bytes) in the binary, it meant I didn't have to worry about shifting code or filling space with NOPs.

<img width="332" height="116" alt="image" src="https://github.com/user-attachments/assets/1f5e4297-520e-401f-a289-8547f1105a5c" />

<img width="196" height="85" alt="image" src="https://github.com/user-attachments/assets/dfe9eac9-7a5d-4c25-bd21-6582cd97d2dd" />

### The Result:
I applied the patches to input file and launched the game. 

The hack worked flawlessly: all the mines were instantly flagged right from the start, and because the user hadn't clicked anything yet, the timer remained safely at 0!

<img width="376" height="273" alt="image" src="https://github.com/user-attachments/assets/73e67840-479e-4e4d-af9c-e0520e17e94a" />

