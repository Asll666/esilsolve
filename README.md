## <img src="https://raw.githubusercontent.com/aemmitt-ns/esilsolve/master/raphi.svg" alt="logo" width="200"/> ESILSolve - A python symbolic execution framework using r2 and ESIL

ESILSolve uses the z3 theorem prover and r2's ESIL intermediate representation to symbolically execute code. 

ESILSolve supports the same architectures as ESIL, including x86, amd64, arm, aarch64 and more (6502, 8051, GameBoy...). This project is a work in progress, though it works well way more often than I honestly thought it would.

### Installation 

To install ESILSolve run `pip install https://github.com/aemmitt-ns/esilsolve/archive/master.zip` or clone it locally and run `pip install .` so you can have the tools directory in a convenient location. 

### Example Usage

```python
from esilsolve import ESILSolver

# start the ESILSolver instance
# and init state with r2 symbol for check function
esilsolver = ESILSolver("test/tests/multibranch")
state = esilsolver.call_state("sym.check")

# make rdi (arg1) symbolic
state.set_symbolic_register("rdi")
rdi = state.registers["rdi"]

# set targets and avoided addresses
# state will contain a state at the target pc addr
state = esilsolver.run(target=0x6a1, avoid=[0x6a8])
print("ARG1: %d " % state.evaluate(rdi).as_long())

```

ESILSolve also easily works with ipa and apk files since they are supported by r2. 

### IPA CrackMe Example

```python
from esilsolve import ESILSolver
import z3 

buf_addr = 0x100000
buf_len = 16

esilsolver = ESILSolver("ipa://tests/crackme-level0-symbols.ipa", debug=False)
state = esilsolver.call_state("sym._validate")
state.registers["x0"] = buf_addr

#use r2pipe like normal in context of the app
validate = esilsolver.r2pipe.cmdj("pdj 1")[0]["offset"]

# initialize symbolic bytes of solution
# and constrain them to be /[a-z ]/
b = [z3.BitVec("b%d" % x, 8) for x in range(buf_len)]
state.constrain_bytes(b, "[a-z ]") 

# concat the bytes and write them to memory 
code = z3.Concat(*b)
state.memory[buf_addr] = code

# success hook callback
def success(state):
    cs = state.evaluate_buffer(code)
    # gives an answer with lots of spaces but it works
    print("CODE: '%s'" % cs.decode())
    esilsolver.terminate()

# set the hooks and run
esilsolver.register_hook(validate+0x210, success)
esilsolver.run(avoid=[validate+0x218, validate+0x3c])
```

### ESILSolve Plugin

The ESILSolve r2 plugin allows the user to quickly use ESILSolve from the r2 console. To add it to r2 run `echo . $(pwd)/tools/esplugin.py >> ~/.radare2rc` 
from the ESILSolve directory. To get the available plugin commands enter `aesx?`. 

```
 -- Are you a wizard?
[0x00000000]> aesx?
Usage: aesx[iscxrebdaw] # Core plugin for ESILSolve
| aesxi[f] [debug] [lazy] [check]              Initialize the ESILSolve instance and VM
| aesxs[bc] reg|addr [name] [length]           Set symbolic value in register or memory
| aesxv reg|addr value                         Set concrete value in register or memory
| aesxc sym value                              Constrain symbol to be value, min, max, regex
| aesxc[+-]                                    Push / pop the constraint context
| aesxx[ec] expr value                         Execute ESIL expression and evaluate/constrain the result
| aesxr[ac] target [avoid x,y,z]               Run symbolic execution until target address, avoiding x,y,z
| aesxf[c]                                     Resume r2frida after symex is finished
| aesxe[j] sym1 [sym2] [...]                   Evaluate symbol in current state
| aesxb[j] sym1 [sym2] [...]                   Evaluate buffer in current state
| aesxd[j] [reg1] [reg2] [...]                 Dump register values / ASTs
| aesxa[f]                                     Apply the [first] state, setting registers and memory
| aesxw[ls] [state number]                     List or set the current states
```

These commands can also be accessed using the shortcut `aesx...` -> `X...`. Bear in mind that other r2 plugins may use X as well though. 

An example of using the plugin to symbolically execute a simple validation function on a 64 bit value in arm64 Android odex code:

```
[0x6f800b09a0]> Xi
[0x6f800b09a0]> Xs x2 flag 8
[0x6f800b09a0]> Xr 0x6f800b0a1c 0x6f800b09c4
[0x6f800b09a0]> Xc x0 1
[0x6f800b09a0]> Xe flag
flag: 3405691582
[0x6f800b09a0]> ?vx 3405691582
0xcafebabe
```

Here the state is initialized at the current location with `Xi`, and the value in register x2 is made symbolic with `Xs x2 flag 8`. Then the function is run until the return at `0x6f800b0a1c`, avoiding a failure branch at `0x6f800b09c4`. If the validation succeeds x0 should be 1 so we constrain it to that value with `Xc x0 1`. Then the value of the flag is evaluated, and it turns out to be 0xcafebabe.

Some other cool uses of the plugin can be seen in the r2con2020 slides in /docs. Its also easy to make your own ESILSolve based r2 scripts. `ESILSolver()` without arguments will automatically use the current session when the script is run from r2, just like `r2pipe.open()`. Its simple to make powerful, generic tools utilizing symbolic execution. 

### r2frida Integration

ESILSolve works well with any IO plugin for radare2, but it has some special features for r2frida. The `ESILSolver` class has the `frida_state(addr)` method which allows the user to place a frida hook at the provided address (or symbol name). Once this hook is hit the method will return an initialized state with the exact context of the hooked thread. Then the thread is suspended so that the state can be explored symbolically with ESILSolve without the relevant memory (which ES copies lazily) changing. Once, for instance, a desired state has been reached the required input can be evaluated and written back into real program memory. Then `resume()` can be called on the `ESILSolver` instance and the application will continue. An example of this with the same iOS app as above can be seen here 

```python
from esilsolve import ESILSolver
import z3

# start the ESILSolver instance by attaching r2frida 
esilsolver = ESILSolver("frida://usb/attach//iOSCrackMe")

validate = esilsolver.r2api.get_address("validate")
# initialize state with context from hook, app is suspended
state = esilsolver.frida_state(validate)

# initialize symbolic bytes of solution
# and constrain them to be /[a-zA-Z]/
code = z3.BitVec("code", 16*8)
state.constrain_bytes(code, "[a-zA-Z]")
addr = state.registers["A0"].as_long()
state.memory[addr] = code

state = esilsolver.run(validate+0x210, avoid=[validate+0x218])
solution = state.evaluate_buffer(code)
print("CODE: '%s'" % solution.decode())

# write solution into proper place
# esilsolver.r2api.write(addr, solution) 
esilsolver.resume() # resume suspended app
```

### r2ghidra and P-Code support

ESILSolve has preliminary support for executing the ESIL generated with the r2ghidra command `pdga`. This command translates P-Code into ESIL so that r2 can analyze binaries similarly to Ghidra. It is based on the great work of FX Ti and allows ESILSolve to work with more architectures and support floating point instructions. To use the P-Code translation simply pass `pcode=true` when initializing an `ESILSolver` instance. See `test/float.py` for an example. 

### Docs

This README is getting long and should probably be turned into proper docs. I will do that. Later.