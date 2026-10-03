# Bet365 VM Static Disassembler

> **Disclaimer:** This project is for educational and research purposes only. Use of this tool must comply with the target website's Terms of Service and applicable data privacy laws.

Static disassembler for the **BET365 VM** that transforms encoded VM bytecode into a human-readable representation of its instructions.

---

## Reverse Engineering Approach

We start from the assumption that a JavaScript Virtual Machine (JSVM) can be reduced to a structure similar to the following and nothing here is invented; all the information we needed was obtained by debugging directly in the browser the javascript code:

```javascript
var base64Bytecode = "bigStringBase64Encoded...";

function VM(bytecode) {
    var pc = 0;

    while (true) {
        var opcode = bytecode[pc++];

        switch (opcode) {
            case 0:
                // handler 0 logic ...

            case 1:
                // handler 1 logic ...

            case 2:
                // handler 2 logic ...

            // rest of the handlers logic ...
        }
    }
}

function decoder(base64Bytecode) {
    return [];
}

var bytecode = decoder(base64Bytecode);

VM(bytecode);
```

The exact implementation will vary between JSVMs, but the general architecture is often based on a **bytecode decoder**, a **program counter (`pc`)**, and an **interpreter loop** that dispatches instructions according to their opcode.

Nothing here is invented; all the information we needed was obtained by debugging directly in the browser the javascript code.

### Step 1 — Identify the Bytecode Decoder

The first step is to identify the function responsible for decoding the embedded Base64 bytecode.

The goal is to determine:

* Where the Base64 string is decoded.
* What data structure is produced by the decoder.
* Whether the result is an `Array`, `Uint8Array`, or another buffer-like structure.
* The length and contents of the resulting bytecode.
* Where the decoded bytecode is passed to the VM.

Conceptually:

```text
Base64 string
      ↓
   decoder()
      ↓
 decoded bytecode
      ↓
    VM()
```

### Step 2 — Identify the Program Counter

Once the decoded bytecode is located, the next step is to identify the VM's **program counter (`pc`)**.

A typical interpreter fetches an opcode using a pattern such as:

```javascript
var opcode = bytecode[pc++];
```

The `pc` determines which byte is currently being interpreted and how the VM advances through the bytecode.

### Step 3 — Identify the Opcodes
Once the decoder and the program counter (pc) have been located, the next step is to identify all the opcodes and their corresponding handlers.

In the original client-side VM, opcodes are not dispatched through a switch statement. Instead, the VM uses a dispatch table: an array of functions indexed by the numeric opcode. Each entry in the array is the handler for that opcode.

For example, the original code looks like this:

```javascript
_0xcb42bb[237] = function () {
  var _0x1c5c8f = _0x555150[_0x1ad981[141]++];
  var _0x33cf58 =
    (_0x555150[_0x1ad981[141]++] << 24) |
    (_0x555150[_0x1ad981[141]++] << 16) |
    (_0x555150[_0x1ad981[141]++] << 8) |
    _0x555150[_0x1ad981[141]++];
  if (_0x1ad981[_0x1c5c8f]) {
    _0x1ad981[141] = _0x33cf58;
  }
};

_0xcb42bb[73] = function () {
  var _0x4a55e6 =
    (_0x555150[_0x1ad981[141]++] << 24) |
    (_0x555150[_0x1ad981[141]++] << 16) |
    (_0x555150[_0x1ad981[141]++] << 8) |
    _0x555150[_0x1ad981[141]++];
  _0x1ad981[141] = _0x4a55e6;
};

_0xcb42bb[83] = function () {
  var _0x456c57 = _0x555150[_0x1ad981[141]++];
  var _0x432349 =
    (_0x555150[_0x1ad981[141]++] << 24) |
    (_0x555150[_0x1ad981[141]++] << 16) |
    (_0x555150[_0x1ad981[141]++] << 8) |
    _0x555150[_0x1ad981[141]++];
  if (!_0x1ad981[_0x456c57]) {
    _0x1ad981[141] = _0x432349;
  }
};
```
Where:

_0xcb42bb is the dispatch table (array of handler functions).

_0x555150 is the decoded bytecode buffer.

_0x1ad981[141] is the program counter (pc).

_0x1ad981 is the register array (VM state).

This means the VM's main loop is conceptually:

```javascript
while (true) {
  var opcode = bytecode[pc++];
  dispatchTable[opcode]();   // execute the handler for this opcode
}
```
Since working with numeric indices into an array of anonymous functions is hard to read, we converted the dispatch table into a switch statement, mapping each numeric opcode to a named case:

```javascript
switch (opcode) {
  case 73:  // JMP
  case 83:  // JMP_IF_FALSE
  case 237: // JMP_IF_TRUE
  case 144: // MOV
  case 80:  // NEW
  case 67:  // CLOSURE
  case 188: // CALL_FUNC
  case 172: // GETPROP
  case 56:  // SETPROP
  // ...
}
```
This conversion does not change the semantics — it is purely a readability improvement. Each case corresponds exactly to one entry in the original dispatch table.

This process produces the opcode table where each entry records:

The numeric opcode value.

A human-readable mnemonic (e.g., MOV, JMP, CALL_FUNC).

The handler logic (what the VM does when executing that opcode).

The number and type of operands it consumes.

Some opcodes may appear duplicated or share similar names (e.g., LESS THAN for both 77 and 244). These are either aliases, encoding quirks, or errors in the original VM. They must be verified against the actual execution.


### Step 4 — Determine Operand Consumption

For every opcode handler, we need to determine exactly how many bytes are consumed from the bytecode, also debugging.

For example:

```javascript
case 144: {
    var dst = bytecode[pc++];
    var src = bytecode[pc++];

    registers[dst] = registers[src];
    break;
}
```

This tells us that opcode `144` has the following bytecode layout:

```text
[opcode] [dst] [src]
```

and consumes:

```text
1 opcode byte
2 operand bytes
----------------
3 bytes total
```

For variable-length instructions, the number of consumed bytes may depend on an operand:

```javascript
case 80: {
    var dst = bytecode[pc++];
    var ctor = bytecode[pc++];
    var argc = bytecode[pc++];

    for (var i = 0; i < argc; i++) {
        var arg = bytecode[pc++];
    }

    break;
}
```


To read multi-byte operands, we use the readInt32 function, which reads a 32-bit big-endian integer from the bytecode and advances the program counter:

```javascript
function readInt32(bytecode, state) {
  return (
    (bytecode[state.pc++] << 24) |
    (bytecode[state.pc++] << 16) |
    (bytecode[state.pc++] << 8) |
    bytecode[state.pc++]
  );
}
```
This function is used by opcodes such as 67 (CLOSURE), 73 (JMP), 83 (JMP_IF_FALSE), and many others that contain 32-bit addresses or integers.

All of this information was extracted directly from the client-side code of the VM. Nothing was invented; every operand layout and consumption rule was derived by debugging the actual execution and reading the original implementation.


### Step 5 — Build the Disassembler

Once the operand layout of each opcode is understood, we can reconstruct the bytecode instruction set and implement a static disassembler.

The process becomes:

```text
Base64 bytecode
       ↓
   Decoder analysis
       ↓
   Identify VM loop
       ↓
   Identify PC
       ↓
   Identify opcodes
       ↓
   Determine operand layout
       ↓
   Determine instruction semantics
       ↓
    Disassembler
```

The key objective is to build a mapping between the raw bytecode representation and a human-readable instruction representation:

```text
Raw bytecode:

[144] [5] [12]

        ↓

Disassembly:

[0x0000] MOV R5, R12
```

By recovering the operand consumption and semantics of each opcode, the VM's bytecode can be progressively transformed from a raw binary representation into a readable instruction stream suitable for static analysis and further reverse engineering.


# 5. Opcode table

| Opcode | Instruction            |
| -----: | ---------------------- |
|    `1` | `LOAD_STR`             |
|    `9` | `LESS OR EQUAL`        |
|   `21` | `DIV`                  |
|   `25` | `THROW`                |
|   `26` | `RIGHTSHIFT` (`>>>`)   |
|   `35` | `LOAD_FLOAT`           |
|   `36` | `STRICT EQUAL`         |
|   `55` | `RET`                  |
|   `56` | `SETPROP`              |
|   `66` | `SUB`                  |
|   `67` | `FUNC`                 |
|   `73` | `JMP`                  |
|   `77` | `LESS THAN`            |
|   `80` | `NEW`                  |
|   `83` | `JMP_IF_FALSE`         |
|   `84` | `LESS OR EQUAL`        |
|   `93` | `ADD`                  |
|  `115` | `LOAD_BYTE`            |
|  `144` | `MOV`                  |
|  `150` | `EXIT`                 |
|  `154` | `ARRAY_CREATE`         |
|  `156` | `SHIFTRIGHT` (`>>`)    |
|  `161` | `MUL`                  |
|  `169` | `TRY_CATCH_FINALLY`    |
|  `172` | `GETPROP`              |
|  `188` | `CALL_FUNC`            |
|  `198` | `CALL`                 |
|  `204` | `AND`                  |
|  `220` | `OR`                   |
|  `228` | `EQUAL` (`==`)         |
|  `231` | `LOAD_INT`             |
|  `235` | `MODULE`               |
|  `237` | `JMP_IF_TRUE`          |
|  `244` | `LESS THAN`            |
|  `247` | `XOR`                  |
|  `248` | `LSHIFT`               |
|  `254` | `CMPNE` (`!=`)         |
|  `255` | `STRICT EQUAL` (`===`) |



