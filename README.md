# ESILSolve - A python symbolic execution framework using r2 and ESIL

ESILSolve uses the z3 theorem prover and r2's ESIL intermediate representation to symbolically execute code. 

ESILSolve supports the same architectures as ESIL, including x86, amd64, arm, aarch64 and more. This project is a work in progress.

Example Usage

```python
from esilsolve import ESILSolver
import r2pipe

# open an r2pipe with the target binary
# and init the esil vm
r2p = r2pipe.open("tests/multibranch", flags=["-2"])
r2p.cmd("s sym.check; aei; aeim;")

#start the ESILSolver instance
esilsolver = ESILSolver(r2p, debug=False, trace=False)
state = esilsolver.init_state()

# make rdi (arg1) symbolic
state.set_symbolic_register("rdi")
rdi = state.registers["rdi"]

# hook callback
def success(instr, state):
    sat = state.solver.check()
    m = state.solver.model()
    print("ARG1: %d" % m.eval(rdi, True).as_long())
    return True

# hook any address to manipulate states
# and set targets and avoided paths
esilsolver.register_hook(0x6a1, success)
esilsolver.run(target=0x6a1, avoid=[0x6a8])
```