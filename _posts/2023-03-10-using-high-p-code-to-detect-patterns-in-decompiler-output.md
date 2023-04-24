# Ghidra: using high P-code to detect patterns in decompiler output

As part of the [_Metal Gear Solid_ matching decompilation](https://github.com/FoxdieTeam/mgs_reversing) effort (ie, reverse-engineering the game's binaries to create C code which compiles to byte-for-byte identical output), I wanted to create a Ghidra script that would detect usage by the original developers of some of the C macros provided by the PlayStation SDK, PsyQ. Was this possible? If so, how?

## Table of contents
- [Background: the PlayStation SDK's C macros](#background)
- [The problem: hunting for macro usage](#problem)
- [The solution: high P-code](#solution)
- [A plugin to make high P-code more easily accessible](#plugin)

## Background: the PlayStation SDK's C macros <a name="background"></a>

The transition to the fifth generation of video game consoles in the 1990s (3DO, Saturn, PlayStation, N64 ...) gave developers revolutionary new capabilities in terms of real-time 3D graphics, which simultaneously brought with them a new level of complexity that the console manufacturers sought to alleviate by providing C libraries for the first time. To developers working with the PlayStation, Sony offered the PsyQ SDK, with an API that they could use to work with the Geometry Transformation Engine (coprocessor for high-speed geometry operations), memory cards, the CD-ROM drive, and other aspects of the system.

To work with the GPU, developers could use libgpu, which allowed them to prepare primitives such as polygons, lines and sprites and put them in ordering tables to be sent to the GPU for drawing. A flat-shaded triangle, for instance, was defined as
```c
typedef struct
{
    u_long tag;
    u_char r0, g0, b0, code;
    short  x0, y0;
    short  x1, y1;
    short  x2, y2;
} POLY_F3;
```
while a 16-by-16 sprite was defined as
```c
typedef struct
{
    u_long  tag;
    u_char  r0, g0, b0, code;
    short   x0, y0;
    u_char  u0, v0;
    u_short clut;
} SPRT_16;
```
and so on and so forth. Note that all primitives have an identically defined header; where inheritance might have been used if this had been C++, in this C code primitives were instead accessed in a "type-agnostic" way by casting them to this generic struct:
```c
typedef struct
{
    unsigned addr : 24;
    unsigned len : 8;
    u_char   r0, g0, b0, code;
} P_TAG;
```
Generic primitive-handling macros were then defined in terms of this struct:
```c
#define setlen( p, _len) 	(((P_TAG *)(p))->len  = (u_char)(_len))
#define setaddr(p, _addr)	(((P_TAG *)(p))->addr = (u_long)(_addr))
#define setcode(p, _code)	(((P_TAG *)(p))->code = (u_char)(_code))
```
These were in turn used in macros to help initialize primitives so that the hardware could identify them. For example, the macro
```c
#define setPolyF3(p)	setlen(p, 4),  setcode(p, 0x20)
```
enabled the developer to initialize a flat-shaded triangle simply by writing:
```c
POLY_F3 p;
setPolyF3(&p);
```

## The problem: hunting for macro usage <a name="problem"></a>

So how might one go about automating the task of finding out when the developers used these macros and highlighting such cases with helpful comments indicating which macro was used? In Ghidra's decompiler output, it is relatively easy to make an educated guess about this. A line such as the following stands out as a good candidate, assigning `0x20` (the `POLY_F3` code) to byte 7 of some struct (precisely the same offset as that of the `code` field in the primitive header):
```c
*(undefined *)((int)puVar5 + 7) = 0x20;
```
This instantly poses the problem of scriptability; it is clear that we need to work with a more low-level representation of the code if we are to achieve our goals. Perhaps the answer lies in the intermediate representation that Ghidra generates from the assembly code, that is, [P-code](https://spinsel.dev/assets/2020-06-17-ghidra-brainfuck-processor-1/ghidra_docs/language_spec/html/pcoderef.html)? This representation can easily be displayed for each assembly instruction in the Listing window and could be the solution to our problem. So let us look at the assembly and P-code for the above line:
```asm
sb v0,0x7(t0)
```
<details>
  <summary>View P-code</summary>
  
  ```
  $U100:4 = INT_ADD t0, 7:4
  $U180:4 = COPY 0:4
  $U180:4 = COPY $U100:4
  STORE ram($U180:4), v0:1
  ```
</details>

In MIPS assembly, this instruction means "take the byte stored in register `v0` and store it at a 7-byte offset into the struct pointed to by `t0`". What at once becomes apparent is that we have gone from a representation that holds our desired semantics but is too high-level to work with — the decompiler's pseudo-C output — to one that is systematic and usable for scripting but has the opposite problem of being too low-level: indeed, we have lost the fundamental connection that we seek between the value being assigned (`0x20`) and the offset at which it is being stored (byte 7).

We are naturally tempted to ask: given any registers `rx` and `ry` and the instruction
```asm
sb rx,0x7(ry)
```
can we not simply retrieve the value held in `rx` at that point in function execution? Theoretically perhaps, but in practice this is an extremely impractical endeavour. The instruction we would be looking for,
```asm
li rx,0x20
```
can precede the assignment by an arbitary number of instructions, which might themselves use the same register for other purposes in branches. We would essentially have to write a script that works through the function's data and control flow just to have the register value at that point, which almost sounds like running [P-code emulation](https://medium.com/@cetfor/emulating-ghidras-pcode-why-how-dd736d22dfb) on the whole program. But the decompiler has already done the job of working through the function to resolve constant assignments; is there really no way to access its internal representation of that information programmatically?

## The solution: high P-code <a name="solution"></a>

As it happens, there is. Once Ghidra has executed control and data flow analysis, the decompiler holds a [high-level P-code](https://youtu.be/Qift_6-3y3A?t=1403) which is the final intermediate representation before the generation of pseudo-C, in a process that can be thought of as follows:

![image](https://user-images.githubusercontent.com/77587819/224443557-d52d18f4-2f09-4beb-8928-72e4d6fad615.png)

This high P-code combines the programmatic usability of low P-code with the semantics of the pseudo-C and, as such, is exactly the level at which we want to operate. We can access it via the API thus:
```java
DecompileOptions options = new DecompileOptions();
DecompInterface ifc = new DecompInterface();
ifc.setOptions(options);
// You may or may not want to change the simplification style, see:
// https://ghidra.re/ghidra_docs/api/ghidra/app/decompiler/DecompInterface.html#setSimplificationStyle(java.lang.String)
// ifc.setSimplificationStyle("normalize");
if (!ifc.openProgram(program)) {
	throw new DecompileException("Fatal error", "The decompiler was unable to open the program.");
}
DecompileResults decompResults = ifc.decompileFunction(function, 30, null);
HighFunction highFunction = decompResults.getHighFunction();
Iterator<PcodeOpAST> ast = highFunction.getPcodeOps();
```

Continuing with the above example, the high P-code output for this assignment is
```
(unique, 0x10000118, 4) INT_ADD (unique, 0x10000114, 4) , (const, 0x7, 4)
---  STORE (const, 0x1a1, 4) , (unique, 0x100, 4) , (const, 0x20, 1)
```
It's two operations, but unlike the two assembly instructions, they are always in very close proximity — in fact, it would appear, nearly always in immediate succession! Each operation takes inputs and can produce an output in the form of `Varnode`s, which are triples consisting of an address space, an offset and a size: as an illustration, the first operation, `INT_ADD`, has two inputs,
```
(unique, 0x10000114, 4)
```
and
```
(const, 0x7, 4)
```
(the "offset" is `0x7` here), and produces an output,
```
(unique, 0x10000118, 4)
```
(The [P-code reference manual](https://spinsel.dev/assets/2020-06-17-ghidra-brainfuck-processor-1/ghidra_docs/language_spec/html/pcoderef.html) gives the details for each operation.)

We can access a `PcodeOpAST`'s inputs using `getInput(int index)` to obtain `Varnode`s on which we can call `getOffset()` — not exactly the trickiest part of Ghidra's API. We now have what we need to automate the detection of primitive initialization:
```java
CodeManager codeManager = ((ProgramDB) program).getCodeManager();
FunctionManager functionManager = program.getFunctionManager();

for (Function function : functionManager.getFunctions(startAddr, true)) {
	
	DecompileResults decompResults = ifc.decompileFunction(function, 30, null);
	HighFunction highFunction = decompResults.getHighFunction();
	Iterator<PcodeOpAST> ast = highFunction.getPcodeOps();
	ArrayList<PcodeOpAST> PcodeOps = new ArrayList<PcodeOpAST>();
	while (ast.hasNext()) {
		PcodeOps.add(ast.next());
	}

	for (int i = 0; i < PcodeOps.size(); i++) {

		PcodeOpAST value = PcodeOps.get(i);
		int opCode = value.getOpcode();

		if (opCode == PcodeOp.INT_ADD) {

			long offset = value.getInput(1).getOffset();
			// The primitive code is at a 7-byte offset but compiler optimization of
			// primitive initialization in loops leads to a variety of odd offsets being
			// used.
			if (offset % 2 == 1
				&& i + 1 < PcodeOps.size()) {

				value = PcodeOps.get(i + 1);
				if (value.getOpcode() == PcodeOp.STORE
					&& value.getInput(2).isConstant()) {

					long primCode = value.getInput(2).getOffset();
					// primCodeMap is a Map<Integer, String> that eg maps 0x20 to "setPolyF3()".
					if (primCodeMap.containsKey((int) primCode)) {

						String confidence = (offset == 7) ? "Probable" : "Possible";
						String macro = primCodeMap.get((int) primCode);
						// SetComment() is a function defined elsewhere that adds a pre-comment.
						SetComment(value, confidence + " PsyQ macro: " + macro, codeManager);
					}
				}
			}
		}
	}
}
```
This has more nesting than a flock of birds, but it does the job nicely. Using this high P-code, it in fact becomes feasible to search for much more complex patterns than this simple assignment. Take, for instance, the macro that adds a primitive to an ordering table (a linked list of primitives to be sent to the GPU for drawing, called "ordering" because [in the absence of a depth buffer](https://www.copetti.org/writings/consoles/playstation/#drawing-the-scene), it was the programmer's responsibility to ensure things were drawn from furthest to nearest):
```c
#define setaddr(p, _addr)	(((P_TAG *)(p))->addr = (u_long)(_addr))
#define getaddr(p)   		(u_long)(((P_TAG *)(p))->addr)
#define addPrim(ot, p)		setaddr(p, getaddr(ot)), setaddr(ot, p)
```
In Ghidra's pseudo-C, this usually takes on the form:
```c
*(uint *)prim = *prim & 0xff000000 | *ot & 0xffffff;
*(uint *)ot = *ot & 0xff000000 | (uint)prim & 0xffffff;
```
Even restricting ourselves to the first of these two lines (which is sufficient), good luck detecting the pattern with low P-code. Using high P-code, however, this actually becomes feasible:
```java
if (opCode == PcodeOp.INT_AND
		&& value.getInput(0).isRegister()) {

	long offset = value.getInput(1).getOffset();
	long register1 = value.getInput(0).getOffset();
	if (i + 1 < PcodeOps.size()) {

		value = PcodeOps.get(i + 1);
		if (offset == 0xff000000L
				&& value.getOpcode() == PcodeOp.INT_AND
				&& value.getInput(0).isRegister()
				&& value.getInput(1).getOffset() == 0xffffff) {

			long register2 = value.getInput(0).getOffset();
			value = PcodeOps.get(i + 2);
			if (value.getOpcode() == PcodeOp.INT_OR
					&& value.getInput(0).getOffset() == register1
					&& value.getInput(1).getOffset() == register2) {

				SetComment(value, "Probable PsyQ macro: addPrim().", codeManager);
			}
		}
	}
}
```

With some more work, we get the following results:

![image](https://user-images.githubusercontent.com/77587819/224443586-c9b90e0e-aaa3-4a29-bf96-82ae5288b948.png)
![image](https://user-images.githubusercontent.com/77587819/224443595-8c569d9f-75db-471a-a719-fa048e95756e.png)

## A plugin to make high P-code more easily accessible <a name="plugin"></a>

One hurdle I bumped into when writing the pattern detection algorithm was that Ghidra does not provide a convenient way to view the high P-code for a function. There are two ways to do view high P-code from within the UI, both with notable shortcomings.
- [One possibility](https://reverseengineering.stackexchange.com/a/27202) is to run `currentLocation.token.pcodeOp` inside a Python shell or script, which will return the high P-code operation for the last token clicked on in the Decompile window; however, the token may only be a part of the larger expression that we actually want to see the high P-code of, and trying to scale this to the level of complexity we need would result in a very subpar user experience.
- [Another](https://reverseengineering.stackexchange.com/a/29652) is to use the Decompile window's Menu (dropdown arrow) to access the Graph Control Flow window, which is meant to graph the function's control flow (you don't say!) rather than show the high P-code per se and is, as such, an extremely limited and clunky way to try to view the function's high P-code. In fact, no matter the Program Graph options, I couldn't get it to display more than 10 lines of high P-code for a given code path.

For that reason, I developed a plugin which makes viewing the high P-code a much more convenient and pleasant experience, showing the high P-code for the whole function but highlighting and scrolling into focus that which corresponds to the clicked-on pseudo-C statement:

![HighPCodeViewer](https://user-images.githubusercontent.com/77587819/224444631-112f70f1-544e-416c-89c9-13257e11ab53.gif)

It can be downloaded [here](https://github.com/twevs/HighPCodeViewer).
