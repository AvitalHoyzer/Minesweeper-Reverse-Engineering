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

### Tracking the Mouse Click to the Board Address (Static Analysis)

I entered sub_1001BC9 and I noticed it had a really big "switch case" structure inside it.

Since this function is responsible for processing inputs, I knew that whenever a player clicks on the board, the game must access the board's data in memory to see what's there.

So, I started digging through the code inside that function to find where it accesses memory.
I was looking for instructions that access memory (using square brackets [])
After trying a few different paths, I found this specific block:

<img width="281" height="175" alt="image" src="https://github.com/user-attachments/assets/a55944b3-a394-4b7b-9472-9624af3ecd6f" />

This was the breakthrough. The instruction `mov al, byte_1005340[eax+ecx]` was the direct link between the game's logic and the board's memory. By clicking on byte_1005340 in the disassembler, it pointed me to the base address **0x01005340**.

<img width="518" height="113" alt="image" src="https://github.com/user-attachments/assets/c316ad46-6354-40b9-b0a8-2858cccaebf3" />

**How I verified this is the board:**

I didn't know what the data meant, only that this address was being accessed. To prove this was the board, I looked at what the code does after fetching that data:

**Observing the Render Loop:** I noticed that the value fetched from that memory address is passed directly into a chain of instructions that ends with call ds:SetPixel.
 
**Linking Logic to Visuals:** SetPixel is a Windows function that literally paints a single pixel on the screen. Because the code fetches a value from the memory address and immediately uses it to decide which color to paint on the screen, it is undeniable that this memory address contains the "instructions" or the state for the board's visual display.

**Confirmation(Dynamic Analysis):**

I ran the game and opened the memory at 0x01005340 in the Hex View.

The grid structure was instantly visible. I could see the board layout clearly defined by 10h values, which are the game’s internal "walls" or borders.

<img width="460" height="481" alt="image" src="https://github.com/user-attachments/assets/53d479d6-a2a6-452a-a323-22ccb67f43e4" />


To confirm this was the live board data, I kept the Hex View open while playing. I noticed that the values remained static until I interacted with the game, at which point the specific bytes updated in real-time, perfectly mirroring my visual actions on the screen.

<img width="129" height="193" alt="image" src="https://github.com/user-attachments/assets/91f7fc76-66aa-4776-b2ea-201c9e65e8fd" />

<img width="404" height="278" alt="image" src="https://github.com/user-attachments/assets/cb33c58e-503d-41a0-8f64-b9017d7be183" />

## 3. What values can a cell on the board hold? (Dynamic Analysis)

I debugged(It wasn't necessary to put a breakpoint), ran the game, and focused on the board's memory address (0x01005340).

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

To understand the connection between these values and what I saw on the screen, I looked at the code. Each cell is one byte, and the game uses the bits inside that byte to store the status and the image.

### Checking Management Bits (Bit 6 & Bit 7)
Inside the cell-updating function at address **0x01002FA4** (sub_1002F80), we can see how the game evaluates the status flags of each square:

<img width="341" height="206" alt="image" src="https://github.com/user-attachments/assets/6e5088b6-99e0-498f-9b5a-5b0de33fb55d" />

* **Is the cell opened? (Bit 6):** The instruction `test al, 40h` (01000000 in binary) checks if bit 6 is set. If this bit is 1, the cell has been clicked and revealed. if it is 0, the cell remains hidden.

* **Is there a mine? (Bit 7 - The MSB):** Further down in the same block, the program executes a sign-flag check (`test al, al` followed by jns). This natively checks if the highest bit (Bit 7, or 80h / 10000000 in binary) is active, which serves as the "Mine Flag".

### Isolating the Visual Appearance (Bits 0-4)
To decide which "sprite" (image asset) to draw on the screen, the game strips away the management bits to find the visual state. This occurs inside the drawing routine at address 0x0100266F (sub_1002646):

<img width="183" height="47" alt="image" src="https://github.com/user-attachments/assets/46476101-18af-4bf5-a429-7d85ae4ac09d" />

The bitwise mask `and edx, 1Fh` (00011111) isolates the lowest 5 bits.

This resulting value is the index for the icon (from the graphic array hdcSrc):

For example:
  * **0Fh**(00001111): Hidden square.
  * **0Eh**(00001110): Flag.
  * **0Dh**(00001101): Question mark.
  * **00h**(00000000): Empty square.
  * **01h** to **08h**(00000001-00001000): An opened square showing a number(maximum mines around a square is 8).

While analyzing the byte structure, I noted that Bit 5 is included in the byte but is consistently ignored by the rendering function (sub_1002646) due to the 1Fh (00011111) mask. Furthermore, the game logic functions do not explicitly test Bit 5 for state validation. This indicates that Bit 5 is likely a 'reserved' or 'internal-state' bit, potentially used during the initial board-generation phase or for temporary calculations during neighbor-clearing, without directly impacting the visual state of the cell.

## Summary of my findings
* **Hidden Mine**: 8Fh **->** 100 | 01111 **->** Mine bit (80h) + Hidden sprite (0Fh)

* **Flagged Mine**: 8Eh **->** 100 | 01110 **->** Mine bit (80h) + Flag sprite (0Eh)

* **Opened Empty**: 40h **->** 010 | 00000 **->** Opened bit (40h) + Empty sprite (00h)

* **Numbered (in the screenshot - 1)**: 41h **->** 010 | 00001 **->** Opened bit (40h) + Number sprite (01h)

* **Exploded Mine**: CCh **->** 110 | 01100 **->** Mine (80h) + Opened (40h) + Explosion (0Ch)

* **Wrong Flag**: 0Bh **->** 000 | 01011 **->** Default + Wrong Flag sprite (0Bh)

 
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

<img width="122" height="188" alt="image" src="https://github.com/user-attachments/assets/5b8d52e6-7715-4555-aead-de9ee680b0d4" />

<img width="122" height="185" alt="image" src="https://github.com/user-attachments/assets/93520f2c-681b-4f49-82cb-26f8caba7d36" />



## Part B
---
I tried a lot of things until I realized that in the end, it was actually very simple. Here is the full process of my experiments and how I found the final solution, answering the required questions along the way.

### 4,5. Where is the game board size and the number of mines determined?
At first, I looked for where the board size is set. I opened the Strings window in IDA to search for keywords like HEIGHT or WIDTH. Instead, I found these Windows API functions:

* GetPrivateProfileIntW

* GetPrivateProfileStringW

<img width="323" height="29" alt="צילום מסך 2026-06-20 234636" src="https://github.com/user-attachments/assets/f47fbb6e-fe89-40c5-8844-5a1a2b0853c7" />


These APIs are used to read configuration files (like .ini files). I pressed Cross-References (X) on GetPrivateProfileIntW to see where it is used, and it brought me to function sub_1003A12.

<img width="337" height="311" alt="צילום מסך 2026-06-20 234859" src="https://github.com/user-attachments/assets/b6126e6f-0af7-44d0-8135-a69cbbad453b" />

This function is a wrapper that tries to load a file named entpack.ini into the EBX register. The key line here is:
`lea eax, ds:10050D0h[eax*4]`

<img width="472" height="280" alt="צילום מסך 2026-06-20 235118" src="https://github.com/user-attachments/assets/8318256b-ff60-4ffa-87af-bfe04e765fea" />

<img width="500" height="88" alt="צילום מסך 2026-06-20 235929" src="https://github.com/user-attachments/assets/32f7b724-1e65-461f-9a73-7fb30875fa9c" />

This points to an array of strings in memory. Every time the function is called, it grabs a different string to use as a setting name (like "Height", "Width", and "Mines"). Since there is no entpack.ini file in the game folder, the program is supposed to fall back to hardcoded default values.

I checked the Cross-References (X) for where sub_1003A12 is called, and found it inside function sub_1003AB0 in a code block named loc_1003B68.

<img width="215" height="316" alt="צילום מסך 2026-06-21 001249" src="https://github.com/user-attachments/assets/ca38ef42-54a7-4a49-bfda-b8a27ae3448b" />


I placed a breakpoint at the start of sub_1003AB0 and ran the debugger. To my surprise, the code completely skipped this default settings block! It jumped straight to the right-side block. Why? Because the game was already run on this computer before, so its configuration data already existed in the Windows Registry. It didn't need to read the defaults or the INI file.

<img width="559" height="167" alt="צילום מסך 2026-06-21 003633" src="https://github.com/user-attachments/assets/7f8cf1d0-245f-4526-8778-20f5116ff8cf" />

<img width="538" height="269" alt="צילום מסך 2026-06-21 003934" src="https://github.com/user-attachments/assets/322205d9-0765-4626-b181-9a649164f821" />


To find where the code actually pulls the real numbers (like 10 mines), I looked at the right-side block of sub_1003AB0. When the game successfully opens the Registry key.

I opened the Registry Editor on my computer and navigated to the path found in the code:
HKEY_CURRENT_USER\Software\Microsoft\winmine

There I found all the active values the game was using: Mines was set to 10, Height was 16, Width was 16, and Difficulty was 0 (Beginner).

<img width="499" height="401" alt="צילום מסך 2026-06-21 004125" src="https://github.com/user-attachments/assets/926854b2-7cdb-4c5f-a7a9-760054ba20a2" />

### My Initial Experiments (What went wrong):

**Registry Experiment:** I tried changing the Mines value in the Registry directly to 1. But this caused a complete mess on the board (Board Corruption). There were random flags everywhere, sometimes 2 mines, sometimes 1. This happened because of Registry Persistence / State Mismatch. the game saves old session data (like flag and mine positions) from previous games in the Registry, and it was fighting against the new changes.

**Skipping the Registry Experiment:** I realized they asked to "patch" the code, so I shouldn't touch the Registry. I went back to IDA and tried to force the code to run through the left-side default block by changing a jnz instruction to a jmp. I also patched the default mine value from push 0Ah (10) to push 1. But this caused the game to crash with a warning: referenced memory at 0xABABABAC. The memory could not be written. This happened because changing that jump messed up the Stack balance, making the function read uninitialized garbage memory.

**The Menu Realization:** I also realized that changing the default values in sub_1003AB0 wouldn't help if a player changes the difficulty level inside the game menu. When you select Beginner, Intermediate, or Expert, a block inside sub_1001BC9 reads the mine counts directly from a hardcoded memory array (dword_1005010) and updates a variable named dword_10056A4.


### 6. How should the code be patched so that upon running there is only a single mine on the board?

The goal is to win the game as fast as possible (without patching the timer). Instead of fighting the Registry or the menu settings, the perfect solution was right in front of me the whole time: Patch the code at the exact place where the board is freshly generated.

I looked at function sub_100367A. This function is responsible for creating a new game board. It is called on startup, and it is also called every time you click a difficulty level in the menu. Inside this function, it prepares the loop that randomly scatters the mines on the board:

<img width="161" height="137" alt="image" src="https://github.com/user-attachments/assets/7e335bf1-536a-424a-9c10-874945b38e86" />

The variable dword_1005330 is the actual live counter for the mine-placement loop. If we change EAX to 1 right before this variable is set, the loop will place exactly 1 mine and stop.

<img width="165" height="139" alt="image" src="https://github.com/user-attachments/assets/89528df1-12c7-4b72-a81b-dc3004419ddc" />

### Result:
Now, no matter what difficulty level is chosen, and no matter what is written in the Registry, the placement loop is forced to receive a value of 1. The game generates a perfectly clean board of the correct size but puts only one mine on it. Clicking any safe square instantly triggers a victory because all non-mine squares are revealed at once!

<img width="374" height="271" alt="image" src="https://github.com/user-attachments/assets/a316d005-b8cb-4c5a-958d-c14d8382fca5" />


