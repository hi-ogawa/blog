<!--
{
  "title": "V8: Ignition Turbofan",
  "date": "2017-03-25T01:54:33+09:00",
  "category": "",
  "tags": ["v8", "javascript", "source"],
  "draft": false
}
-->

It seems there's a long history the way V8 runs Javascript, but happily
the code base I see now only needs Ignition (interpreter/) and Turbofan (compiler/).

# Summery

- v8 api from v8_shell
- javascript to ignition bytecode
- ignition bytecode to turbofan. IR
- Turbofan IR to x64 assembly
- code stubs, builtin, runtime paths

# Following samples/shell.cc

```
[ Basic Steps ]
- platform::CreateDefaultPlatform => this spawns 3 WorkerThread s
- InitializePlatform(platform)
- Isolate::New =>
  - new Isolate
  - IsolateNewImpl => Isolate::Init
- (block)
  - Isolate::Scope isolate_scope
  - HandleScope handle_scope
  - (Local) v8::Context::New
  - Context::Scope context_scope
  - (loop for)
    - HandleScope handle_scope
    - (Local) Isolate::GetCurrentContext
    - (Local) Script
    - Script::Compile
    - Script::Run
    - platform::PumpMessageLoop

[ Compilation (Ignition bytecode) ]
- Script::Compile => ScriptCompiler::Compile =>
  - CompileUnboundInternal =>
    - Compiler::GetSharedFunctionInfoForScript (with NON_NATIVES_CODE)
      - Factory::NewScript
      - set Script::Typescript (e.g. TYPE_NATIVE)
      - prepare ParseInfo, Zone, CompilationInfo
      - CompileToplevel =>
        - parsing::ParseProgram =>
          - Parser::ParseProgram =>
            - DoParseProgram (handmade lexer/parser implementation. let's not follow for now)
            - (PrintF when FLAG_trace_parse)
          - ParseInfo::setLiteral
        - NewSharedFunctionInfoForLiteral =>
          - Builtins::CompileLazy (defined by BUILTIN_LIST_ALL(DEFINE_BUILTIN_ACCESSOR))
            - returns address of some executable text (builtins_[kCompileLazy] ?)
          - Factory::NewSharedFunctionInfo =>
            - SharedFunctionInfo::set_code
          - SharedFunctionInfo::InitFromFunctionLiteral
        - CompileUnoptimizedCode =>
          - Compiler::Analyze =>
            - Rewriter::Rewrite (see comment class Rewriter) => Processor::Process (as ASTVisitor)
            - DeclarationScope::Analyze (see comment class Scope) =>
              - AllocateVarialbe =>
                - Scope::ResolveVariablesRecursively (see comment class Variable)
                - AllocateVariablesRecursively
                - AllocateScopeInfosRecursively
            - Renumber
          - CompileUnoptimizedInnerFunctions =>
            - GenerateUnoptimizedCode =>
              - GetUnoptimizedCompilationJob => ... => new InterpreterCompilationJob
              - InterpreterCompilationJob::ExecuteJob => ExecuteJobImpl =>
                - BytecodeGenerator::GenerateBytecode =>
                  - (do its stuff using BytecodeArrayBuilder as AstVisitor, follow this later)
              - FinalizeUnoptimizedCompilationJob =>
                - InterpreterCompilationJob::FinalizeJob => FinalizeJobImpl =>
                  - BytecodeGenerator::FinalizeBytecode (emit BytecodeArray)
                  - (if FLAG_print_bytecode, BytecodeArray::Print, which leads to BytecodeArray::Disassemble)
                  - CompilationInfo::SetBytecodeArray
                  - CompilationInfo::SetCode (builtin's InterpreterEntryTrampoline)
                - InstallUnoptimizedCode
                  - InstallSharedCompilationResult
                    - SharedFunctionInfo::ReplaceCode with CompilationInfo.code
                    - SharedFunctionInfo::set_bytecode_array to CompilationInfo.bytecode_array
    - return UnboundScript with SharedFunctionInfo
  - UnboundScript::BindToCurrentContext =>
    - cast to SharedFunctionInfo
    - Factory::NewFunctionFromSharedFunctionInfo =>
      - NewFunction
      - (returns JSFunction)

[ Execution ]
- Script::Run => Execution::Call => CallInternal => Invoke =>
  - code = Factory::js_entry_code (relavant macros ROOT_ACCESSOR, ROOT_LIST, STRONG_ROOT_LIST)
  - JSEntryFunction stub_entry = Code::entry
  - CALL_GENERATED_CODE (which actually calls stub_entry as normal C function)
    - cannot track debug symbol ??

[ Data Structure ]
CompilationInfo
'-' ParseInfo
  '-' Script (extends Struct)
    '-' Object (as source string)
    '-' Object (as name of script)
  '-' FunctionLiteral (AST expression)

SharedFunctionInfo (extends HeapObject)
'-' Code
'-' AbstractCode
'-' ScopeInfo
'-' Script

FunctionLiteral (extends Expression)
'-' AstString (as raw_name_)
'-* Statement (as body_)
'-' DeclarationScope

JSFunction (extends JSObject)
'-' SharedFunctionInfo
```


# Code generation details

- Bytecode generation

```
## Example is `1 + 1 === 1 + 1 ? 2 : 3` ##

[ AST printed within debugger ]
(lldb) expr stmt->Print()
EXPRESSION STATEMENT at 0
. ASSIGN at -1
. . VAR PROXY local[0] (mode = TEMPORARY) ".result"
. . CONDITIONAL at 0
. . . CONDITION at 6
. . . . EQ_STRICT at 6
. . . . . LITERAL 2
. . . . . LITERAL 2
. . . THEN at 18
. . . . LITERAL 2
. . . ELSE at 22
. . . . LITERAL 3

[ Trace from --print_bytecode ]
Parameter count 1
Frame size 16
    0 E> 0x8a5e05abda6 @    0 : 88                StackCheck
    0 S> 0x8a5e05abda7 @    1 : 03 02             LdaSmi [2]
         0x8a5e05abda9 @    3 : 1f f9             Star r1
         0x8a5e05abdab @    5 : 03 02             LdaSmi [2]
    6 E> 0x8a5e05abdad @    7 : 56 f9 02          TestEqualStrict r1, [2]
         0x8a5e05abdb0 @   10 : 7f 06             JumpIfFalse [6] (0x8a5e05abdb6 @ 16)
         0x8a5e05abdb2 @   12 : 03 02             LdaSmi [2]
         0x8a5e05abdb4 @   14 : 72 04             Jump [4] (0x8a5e05abdb8 @ 18)
         0x8a5e05abdb6 @   16 : 03 03             LdaSmi [3]
         0x8a5e05abdb8 @   18 : 1f fa             Star r0
   23 S> 0x8a5e05abdba @   20 : 8c                Return

[ BytecodeGenerator path ]
- BytecodeGenerator::GenerateBytecode =>
  - GenerateBytecodeBody =>
    - VisitStatements =>
      - Visit (macro from DEFINE_AST_VISITOR_SUBCLASS_MEMBERS) => ... =>
        - VisitConditional =>
          - (check if we can skip conditional by ToBooleanIsTrue and ToBooleanIsFalse, but curren case, it didn't reduce that.)
          - VisitForTest =>
            - Visit => VisitCompareOperation =>
              - VisitForRegisterValue (for lhs `2` (actually `1 + 1` is folded somewhere)) =>
                - VisitForAccumulatorValue =>
                  - ValueResultScope::ValueResultScope (pair with ~ValueResultScope)
                  - Visit => VisitLiteral =>
                    - BytecodeArrayBuilder::LoadLiteral =>
                      - AstValue::AsSmi
                      - LoadLiteral => OutputLdaSmi (defined from BYTECODE_LIST(DEFINE_BYTECODE_OUTPUT)) =>
                        - BytecodeNodeBuilder<kOutputLdaSmi>::Make =>
                          - (some optimization called BytecodeRegisterOptimizer::PrepareOutputRegister, for now ignore it)
                          - BytecodeNote::Create =>
                        - (optimization opportunities along Write bytecode)
                          - BytecodePeepholeOptimizer::Write => ... => DefautAction =>
                          - BytecodeDeadCodeOptimizer::Write (PeepholeOptimizer keeps and passes last node which is `StackCheck`) =>
                          - BytecodeArrayWriter::Write => EmitBytecode =>
                            - vector::push_back(Bytecodes::ToByte(bytecode))
                  - ValueResultScope::~ValueResultScope =>
                    - BytecodeGenerator::set_execution_result
                - BytecodeRegisterAllocator::NewRegister => Register
                - BytecodeRegisterOptimizer::DoStar => RegisterTransfer (kinda going too far. ignore it for now...)
                - (return Register)
              - VisitForAccumulatorValue (for rhs) => (... similar pattern as above, except crazy optimization stages)
              - CompareOperationFeedbackSlot
              - BytecodeArrayBuilder::CompareOperation => OutputTestEqualStrict => ...
            - BytecodeArrayBuilder::JumpIfFalse (to "else" label) => OutputJumpIfToBooleanFalse
          - (bind "then" label to builder (I guess this is for jump offset calculation happens later ?))
          - VisitForAccumulatorValue ("then" expr) => ...
          - BytecodeArrayBuilder::Jump (to "end" label)
          - (bind "else" label to builder)
          - VisitForAccumulatorValue ("else" expr) => ...
          - (bind "end" label to builder)

```

- Bytecode handler generation (Single bytecode operation => Turbofan IR => x64)

```
[Initilization (compile builtins and bytecode handlers)]
- Isolate::Init =>
  - Builtins::Setup =>
    - (for each builtin code (e.g. InterpreterEntryTrampoline))
      - BuildWithMacroAssembler =>
        - Builtins::Generate_InterpreterEntryTrampoline (as MacroAssemblerGenerator, see below for detail)
        - MacroAssembler::GetCode
      - assign compiled Code to builtins_[kInterpreterEntryTrampoline]
  - Interpreter::Initialize =>
    - (for each bytecode from BYTECODE_LIST (e.g. Bytecode::kDoLdar))
      - Interpreter::InstallBytecodeHandler(...kDoLdar)
        - GenerateBytecodeHandler
          - InterpreterDispatchDescriptor
          - InterpreterAssembler
          - InterpreterGenerator::DoLdar => (see below)
          - CodeAssembler::GenerateCode => (see below)
          - (if FLAG_trace_ignition_codege, print Code::Disassemble)
        - assign compiled Code entry to dispatch_table_[index]

[ Turbofan IR Nodes generation ]
(for frame layout, see google's ignition design document)
- InterpreterGenerater::DoLdar =>
  - InterpreterAssembler::BytecodeOperandReg (returns Node, which represents register's offset within register file) =>
    - BytecodeSignedOperand => BytecodeOperandSignedByte =>
      - OperandOffset (returns Int64Constant "1" Node) =>
        - CodeAssembler::IntPtrConstant(1) => RawMachineAssembler::IntPtrConstant(1) =>
        - Int64Constant(1) =>
          - CommonOperatorBuider::Int64Constant
          - AddNode =>
            - MakeNode => Graph::NewNodeUnchecked => Node::New => new Node
            - Schedule::AddNode =>
              - BasicBlock::AddNode
      - Load (the beggining address of the whole bytecode array) + (this instruction's offset) + (operand offset "1") =>
        - ... => RawMachineAssembler::Load => AddNode
    - CodeAssembler::ChangeInt32ToIntPtr => RawMachineAssembler::ChangeInt32ToInt64 (small x64 addressing fix ?)
  - InterpreterAssembler::LoadRegister =>
    - GetInterpretedFramePointer (create Node representing pointer to trampoline's frame) =>
      - LoadParentFramePointer => ...
    - RegisterFrameOffset => WordShl
    - Load (address of trampoline's frame) + (register's index * 8)
  - InterpreterAssembler::SetAccumulator =>
    - CodeStubAssembler::Variable::Bind (assign loaded register to InterpreterAssembler.accumulator ?)
      - CodeAssemblerVariable::Impl.value (this kind of thing should be represented as edge in turbofan ?)
  - InterpreterAssembler::Dispatch =>
    - Advance =>
      - (emit special thing if --trace_ignition)
      - (assign next bytecode to bytecode_offset_)
      - (returns Node representing next bytecode's offset)
    - LoadBytecode => Load
    - DispatchToBytecode =>
      - (emit special thing if --trace_ignition_dispatches)
      - Load (DispatchTableRawPointer) + (target_bytecode * 8) (this will load next bytecode's trampoline ?)
      - DispatchToBytecodeHandlerEntry (with trampoline and next bytecode offset) =>
        - TailCallBytecodeDispatch (with trampoline, next bytecode offset, accumulator, bytecode array, and dispatch table) =>
          - (arguments except trampoline will be passed to next as it is)
          - Linkage::GetBytecodeDispatchCallDescriptor (?)
          - RawAssembler::TailCallN => MakeNode TailCall, Schedule::AddTailCall

[ Machine code generation ]
- CodeAssembler::GenerateCode =>
  - RawMachineAssembler::Export
  - Pipeline::GenerateCodeForCodeStub =>
    - PipelineImpl::ScheduleAndGenerateCode =>
      - ScheduleAndSelectInstructions =>
        - InitializeInstructionSequence => new InstructionSequence
        - InstructionSelectionPhase::Run =>
          - InstructionSelector::SelectInstructions =>
            - VisitBlock => VisitNode =>
              - (branches by IrOpcode (node._op._opcode) e.g. IrOpcode::kInt64Add)
              - VisitInt64Add (defined instruction-selector-x64.cc) => VisitBionop =>
                - InstructionSelector::Emit(kX64Add) (or EmitDeoptimize?) =>
                  - instructions_.push_back(Instruction::New)
         - AllocateRegister =>
           - (see comment "phase 1" to "phase 9" in register-allocator.h)
           - (here I pick AllocateGeneralRegistersPhase (phase 4))
           - AllocateGeneralRegistersPhase::Run =>
             - LinearScanAllocator::AllocateRegisters =>
               - ( while unhandled_live_ranges is not empty ) =>
                 - ProcessCurrentRange =>
                   - TryAllocateFreeReg (I don't know what "Spliter" and "TopLevel" mean here) =>
                     - find register from RegisterAllocator::allocatable_register_codes
                     - (which originally from ALLOCATABLE_GENERAL_REGISTERS in assembler-x64.h)
      - GenerateCode => GemerateCodePhase::Run => CodeGenerator::GenerateCode =>
        - AssembleBlock => AssembleInstruction =>
          - AssembleArchInstruction (defined in code-generator-x64.cc) =>
            - ArchOpcodeField::decode
            - ASSEMBLE_BINOP(addq) =>
              - Assembler::addq (ASSEMBLER_INSTRUCTION_LIST(DECLARE_INSTRUCTION) in assembler-x64.h) =>
                - emit_add(p1, p2, kInt64Size) => arithmetic_op(0x03, dst, src, size) => emit

(print Instruction (e.g. "v9(R) = X64Movsxbq : MR1I v3(R) v7(R) [immediate:1]"))
- operator<<(std::ostream, PrintableInstruction) =>
  - << PrintableParallelMove => PrintableMoveOperands => PrintableInstructionOperand (PRINT gap ?)
  - << PrintableInstructionOperand                 (PRINT destination operand) =>
    - (branches by InstructionOperand::Kind)
    - InstructionOperand::UNALLOCATED => v<virtual-register-num>(<alocation-policy>) (e.g "v3(R)")
      - Q. what is FIXED_SLOT policy ?
    - InstructionOperand::IMMEDIATE => [immediate:<value>] (e.g. "[immediate:1]")
    - InstructionOperand::ALLOCATED => [<register-name>|R|<size>] (e.g. "[rax|R|w64]")
  - << ArchOpcodeField::decode(instr.opcode())     (PRINT opcode)
  - << AddressingModeField::decode(instr.opcode()) (PRINT addressing mode (macro from TARGET_ADDRESSING_MODE_LIST))
  - << PrintableInstructionOperand                 (PRINT source operands)


[Data structure]
InstructionSequence
'-* InstructionBlock
  '-* Instruction
    '-' InstructionCode
    '-* InstructionOperand (many Kind e.g. UNALLOCATED, CONSTANT, ALLOCATED ...)
    '-* ParallelMove (what's this "gap" thing ?)
      '-* MoveOperands '-2 InstructionOperand (source and destination)

UnallocatedOperatnd (extends InstructionOperand)
'-' Lifetime (lifetime within single instruction)

LiveRange
'-'

$ out.gn/x64.debug/v8_shell --trace_turbo_graph --trace_ignition_codegen # show only Ldar part

Adding #1:Parameter to B0
Adding #2:Parameter to B0
Adding #3:Parameter to B0
Adding #4:Parameter to B0
Adding #5:Parameter to B0
Adding #7:Int64Constant to B0
Adding #8:Int64Add to B0
Adding #9:Load to B0
Adding #10:ChangeInt32ToInt64 to B0
Adding #11:LoadParentFramePointer to B0
Adding #12:Int64Constant to B0
Adding #13:Word64Shl to B0
Adding #14:Load to B0
Adding #15:Int64Constant to B0
Adding #16:Int64Add to B0
Adding #17:Load to B0
Adding #18:ChangeUint32ToUint64 to B0
Adding #19:Int64Constant to B0
Adding #20:Word64Shl to B0
Adding #21:Load to B0
--- RAW SCHEDULE -------------------------------------------
--- BLOCK id:0 ---
  1: Parameter[0](0)    // these parameters are input to interpreter (e.g. accumulator)
  2: Parameter[1](0)    // which also will be passed around next InterpreterDispatch.
  3: Parameter[2](0)
  4: Parameter[3](0)
  5: Parameter[4](0)
  7: Int64Constant[1]
  8: Int64Add(2, 7)
  9: Load[kRepWord8|kTypeInt32](3, 8)
  10: ChangeInt32ToInt64(9)
  11: LoadParentFramePointer
  12: Int64Constant[3]
  13: Word64Shl(10, 12)
  14: Load[kRepTagged|kTypeAny](11, 13)
  15: Int64Constant[2]
  16: Int64Add(2, 15)
  17: Load[kRepWord8|kTypeUint32](3, 16)
  18: ChangeUint32ToUint64(17)
  19: Int64Constant[3]
  20: Word64Shl(18, 19)
  21: Load[kRepWord64](4, 20)
  22: TailCall[Addr:InterpreterDispatch Descriptor:r0s0i5f0t1](21, 14, 16, 3, 4) -> id:1
--- BLOCK id:1 <- id:0 ---
id:0 is not in a loop (depth == 0)
id:1 is not in a loop (depth == 0)
RPO with 0 loops:
    B0:  id:0:
    B1:  id:1:
--- EDGE SPLIT AND PROPAGATED DEFERRED SCHEDULE ------------
--- BLOCK B0 ---
  1: Parameter[0](0)
  2: Parameter[1](0)
  3: Parameter[2](0)
  4: Parameter[3](0)
  5: Parameter[4](0)
  7: Int64Constant[1]
  8: Int64Add(2, 7)
  9: Load[kRepWord8|kTypeInt32](3, 8)
  10: ChangeInt32ToInt64(9)
  11: LoadParentFramePointer
  12: Int64Constant[3]
  13: Word64Shl(10, 12)
  14: Load[kRepTagged|kTypeAny](11, 13)
  15: Int64Constant[2]
  16: Int64Add(2, 15)
  17: Load[kRepWord8|kTypeUint32](3, 16)
  18: ChangeUint32ToUint64(17)
  19: Int64Constant[3]
  20: Word64Shl(18, 19)
  21: Load[kRepWord64](4, 20)
  22: TailCall[Addr:InterpreterDispatch Descriptor:r0s0i5f0t1](21, 14, 16, 3, 4) -> B1
--- BLOCK B1 <- B0 ---
----- Instruction sequence before register allocation -----
IMM#0: 2l
IMM#1: 1l
B0: AO#0 (no frame)  instructions: [0, 11)
 predecessors:
       0: gap () ()
          [r12|R|w64] = ArchNop
       1: gap (v7(-) = [r12|R|w64];) ()
          [r14|R|t] = ArchNop
       2: gap (v3(-) = [r14|R|t];) ()
          [r15|R|w64] = ArchNop
       3: gap (v4(-) = [r15|R|w64];) ()
          v9(R) = X64Movsxbq : MR1I v3(R) v7(R) [immediate:1]
       4: gap () ()
          v8(R) = ArchParentFramePointer
       5: gap () ()
          v1(R) = X64Movq : MR8 v8(R) v9(R)
       6: gap () ()
          v2(R) = X64Lea : MRI v7(R) [immediate:0]
       7: gap () ()
          v6(R) = X64Movzxbl : MR1 v2(R) v3(R)
       8: gap () ()
          v0(R) = X64Movq : MR8 v4(R) v6(R)
       9: gap () ()
          ArchPrepareTailCall
      10: gap () ([rax|R|t] = v1(-); [r12|R|w64] = v2(-); [r14|R|t] = v3(-); [r15|R|w64] = v4(-);)
          ArchTailCallAddress v0(R) [rax|R|t] [r12|R|w64] [r14|R|t] [r15|R|w64] #1
 B1
B1: AO#1 (no frame)  instructions: [11, 12)
 predecessors: B0
      11: gap () ()
          ArchNop

----- Instruction sequence after register allocation -----
IMM#0: 2l
IMM#1: 1l
B0: AO#0 (no frame)  instructions: [0, 11)
 predecessors:
       0: gap () ()
          [r12|R|w64] = ArchNop
       1: gap () ()
          [r14|R|t] = ArchNop
       2: gap () ()
          [r15|R|w64] = ArchNop
       3: gap () ()
          [rax|R|w64] = X64Movsxbq : MR1I [r14|R|t] [r12|R|w64] [immediate:1]
       4: gap () ()
          [rbx|R|w64] = ArchParentFramePointer
       5: gap () ()
          [rax|R|t] = X64Movq : MR8 [rbx|R|w64] [rax|R|w64]
       6: gap () ()
          [r12|R|w64] = X64Lea : MRI [r12|R|w64] [immediate:0]
       7: gap () ()
          [rbx|R|w64] = X64Movzxbl : MR1 [r12|R|w64] [r14|R|t]
       8: gap () ()
          [rbx|R|w64] = X64Movq : MR8 [r15|R|w64] [rbx|R|w64]
       9: gap () ()
          ArchPrepareTailCall
      10: gap () ()
          ArchTailCallAddress [rbx|R|w64] [rax|R|t] [r12|R|w64] [r14|R|t] [r15|R|w64] #1
 B1
B1: AO#1 (no frame)  instructions: [11, 12)
 predecessors: B0
      11: gap () ()
          ArchNop

kind = BYTECODE_HANDLER
name = Ldar
compiler = turbofan
Instructions (size = 68)
0x2bae51d73500     0  4b0fbe442601   REX.W movsxbq rax,[r14+r12*1+0x1]
0x2bae51d73506     6  488bdd         REX.W movq rbx,rbp
0x2bae51d73509     9  488b04c3       REX.W movq rax,[rbx+rax*8]
0x2bae51d7350d     d  4983c402       REX.W addq r12,0x2
0x2bae51d73511    11  430fb61c34     movzxbl rbx,[r12+r14*1]
0x2bae51d73516    16  49ba0000000001000000 REX.W movq r10,0x100000000
0x2bae51d73520    20  4c3bd3         REX.W cmpq r10,rbx
0x2bae51d73523    23  7310           jnc 0x2bae51d73535  <+0x35>
0x2bae51d73525    25  48ba0000000001000000 REX.W movq rdx,0x100000000
0x2bae51d7352f    2f  e82c0ce1ff     call 0x2bae51b84160  (Abort)    ;; code: BUILTIN
0x2bae51d73534    34  cc             int3l
0x2bae51d73535    35  498b1cdf       REX.W movq rbx,[r15+rbx*8]
0x2bae51d73539    39  ffe3           jmp rbx
0x2bae51d7353b    3b  90             nop


Safepoints (size = 8)

RelocInfo (size = 1)
0x2bae51d73530  code target (BUILTIN)  (0x2bae51b84160)

[Except from output when FLAG_trace_alloc case is turned on]
Processing interval 9:0 start=14           // Virtual register v9
Register r12 is free until pos 0 (1)       // but what's about UseInterval.start=14 (looks like 3 * 4 + 2)
Register r14 is free until pos 0 (1)
Register r15 is free until pos 0 (1)
Assigning free reg rax to live range 9:0   // during LinearScanAllocator::TryAllocateFreeReg
Add live range 9:0 to active
```


- Entering runtime function (e.g. Runtime_Compilelazy)
  - runtime.h (FOR_EACH_INTRINSIC)
  - arguments.h (RUNTIME_FUNCTION macro)
  - runtime-compiler.h (RUNTIME_FUNCTION(Runtime_CompileLazy))

- Q.
  - global variable in bytecode (what's with LdarGlobal) or scoping in general
  - meaning of feedback slot use
  - what does CallIC do ?

```
[ Example ]
[ Exsmple: source ]
function fib0(n) {
  if (n <= 1) {
    return n;
  } else {
    return fib0(n - 1) + fib0(n - 2);
  }
}
function main() {
  print(fib0(10)); // => 55
}
main();

[ Example: v8_shell --pring_bytecode ]
Parameter count 2
Frame size 32
   13 E> 0x74567b2c58e @    0 : 88                StackCheck
   21 S> 0x74567b2c58f @    1 : 03 01             LdaSmi [1]
   27 E> 0x74567b2c591 @    3 : 59 02 02          TestLessThanOrEqual a0, [2]
         0x74567b2c594 @    6 : 7f 05             JumpIfFalse [5] (0x74567b2c599 @ 11)
   39 S> 0x74567b2c596 @    8 : 1e 02             Ldar a0
  102 S> 0x74567b2c598 @   10 : 8c                Return
   64 S> 0x74567b2c599 @   11 : 04                LdaUndefined
         0x74567b2c59a @   12 : 1f f9             Star r1
         0x74567b2c59c @   14 : 0a 00 05          LdaGlobal [0], [5]
         0x74567b2c59f @   17 : 1f fa             Star r0
   78 E> 0x74567b2c5a1 @   19 : 38 01 02 07       SubSmi [1], a0, [7]
         0x74567b2c5a5 @   23 : 1f f8             Star r2
   71 E> 0x74567b2c5a7 @   25 : 47 fa f9 f8 03    Call1 r0, r1, r2, [3]
         0x74567b2c5ac @   30 : 1f fa             Star r0
         0x74567b2c5ae @   32 : 04                LdaUndefined
         0x74567b2c5af @   33 : 1f f8             Star r2
         0x74567b2c5b1 @   35 : 0a 00 05          LdaGlobal [0], [5]
         0x74567b2c5b4 @   38 : 1f f9             Star r1
   92 E> 0x74567b2c5b6 @   40 : 38 02 02 0a       SubSmi [2], a0, [10]
         0x74567b2c5ba @   44 : 1f f7             Star r3
   85 E> 0x74567b2c5bc @   46 : 47 f9 f8 f7 08    Call1 r1, r2, r3, [8]
   83 E> 0x74567b2c5c1 @   51 : 2c fa 0b          Add r0, [11]
  102 S> 0x74567b2c5c4 @   54 : 8c                Return
         0x74567b2c5c5 @   55 : 04                LdaUndefined
  102 S> 0x74567b2c5c6 @   56 : 8c                Return
Constant pool (size = 1)
0x74567b2c541: [FixedArray]
 - map = 0x3b48e8982309 <Map(FAST_HOLEY_ELEMENTS)>
 - length: 1
           0: 0x74567b2ba39 <String[4]: fib0>


[ Call1 bytecode generation (AST "Call" -> Call1) ]
- BytecodeGenerator::VisitCall =>
  - (case Call::GetCallType == GLOBAL_CALL (e.g. any outer scope's variable reference ?))
    - get VariableProxy::VariableFeedbackSlot (AstNumberingVisitor::VisitVariableProxy initialize it btw)
    - BuildVariableLoadForAccumulatorValue (emit global variable load for "fib0") =>
      - BuildVariableLoad => (case VariableLocation::UNALLOCATED) LoadGlobal
  - VisitArguments => ...
  - BytecodeArrayBuilder::Call (with Call::CallFeedbackICSlot) => OutputCall1 (macro...)


[ Call1 bytecode handler generation]
(bytecode Call1 -> turbofan IR)
- InterpreterGenerater::DoCall1 => DoJSCallN =>
  - CodeFactory::CallIC =>
    - make_callable => Callable(CallICStub::GetCode)
  - CodeAssembler::CallStubN (with CallICDescriptor) =>
    - Linkage::GetStubCallDescriptor (get CallDescriptor from CallICDescriptor)
    - RawMachineAssembler::CallN (with CallDescriptor) =>
      - CommonOperatorBuilder::Call => IrOpcode::kCall
      - AddNode

(turbofan IR -> arch)
- CodeAssembler::GenerateCode =>
  - (I followed this path above. so, here, I'll only follow specific to IrOpcode::kCall)
  - ... => InstructionSelector::SelectInstructions => VisitNode => (case IrOpcode::kCall) VisitCall =>
    - InitializeCallBuffer =>
      - (case CallDescriptor::kCallCodeObject and kCallCodeImmediate)
        - OperandGenerator::UseImmediate => InstructionSequence::AddImmediate (allocate CallICStub code as immediate)
    - (case CallDescriptor::kCallCodeObject) Emit(kArchCallCodeObject)
  - ... => AssembleArchInstruction =>
    - (case kArchCallCodeObject) MacroAssembler::Call(code, RelocInfo::CODE_TARGET)


[ CallICStub code generation ]
- CodeStub::GetCode (e.g. CallICStub::GetCode) =>
  - (check cache first?)
  - GenerateCode (as TurboFanCodeStub::GenerateCode) =>
    - CallICStub::GenerateAssembly (macro DEFINE_TURBOFAN_CODE_STUB(CallIC, TurboFanCodeStub) and TF_STUB(CallICStub, CodeStubAssembler))
      - ...
      - CodeAssembler::TailCallStub
    - CodeAssembler::GenerateCode
  - FinishCode
  - RecordCodeGeneration
  - Activate


[ Data structure (nats macro...) ]
Callable
'-' CallInterfaceDescriptor

CallICStub (extends TurbofanCodeStub, which extends CodeStub)
'-' CallICDescriptor (extends CallInterfaceDescriptor)
  '-' CallInterfaceDescriptorData
    '-' PlatformInterfaceDescriptor

Linkage
'-' CallDescriptor
  '-' Kind (e.g. kCallCodeObject (e.g. CodeStub), kCallJSFunction)
  '-' LinkageLocation

(in interface-descriptors.h)


[ --trace_ignition_codegen --trace_turbo ]
(Turbofan IR)
49: Call[Code:CallIC Descriptor:r1s2i8f0t0](21, 20, 22, 25, 31, 38, 45, 47)

(Arch Code)
0x26259fadbba1    81  e81af7dcff     call 0x26259f8ab2c0     ;; code: STUB, CallICStub, minor: 6
RelocInfo (size = 14)
0x26259fadbba2  code target (STUB)  (0x26259f8ab2c0)
```


- What's with `Builtin::kCompileLazy` and `Runtime_CompileLazy` ?

`Factory::NewSharedFunctionInfoForLiteral` initialize any function code entry as `CompileLazy` (see above).
Then, this `CompileLazy` is implemented in arch-specificy way via `Builtin::Generate_CompileLazy`,
which calls into C++ implemented `Runtime_CompileLazy` with tail call.


- Compilation of non-top level function during runtime

```
- Runtime_CompileLazy => __RT_impl_CompileLazy => Compiler::Compile =>
  - GetLazyCode => GetUnoptimizedCode =>
    - parsing::ParseAny => ParseFunction (ParseProgram for top-level compilation) =>
      - Parser::ParseFunction
        - (how much is this different from ParseProgram ? formal parameter, outer_scope, arrow function syntax ...?)
    - CompileUnoptimizedCode (we followed this before, see above)
  - JSFunction::ReplaceCode
```


- Turbofan as CompilationJob (only for OSR or FLAG_always_opt ?)

```
[ entries to GetOptimizedCode ]
- Runtime_CompileOptimized_NotConcurrent => Compiler::CompileOptimized => GetOptimizedCode
- Runtime_CompileOptimized_Concurrent => Compiler::CompileOptimized => GetOptimizedCode
- Runtime_CompileForOnStackReplacement => GetOptimizedCodeForOSR => GetOptimizedCode

[ following Compiler::GetOptimizedCode ]
- Compiler::GetOptimizedCode =>
  - (some checks around OSR, cache first?)
  - Pipeline::NewCompilationJob => new PipelineCompilationJob
  - (AbortOptimization for some cases (e.g. SharedFunctionInfo::HasDebugInfo))
  - GetOptimizedCodeNow (let's forget background thread) =>
    - (if not optimizing_from_bytecode, Compiler::ParseAndAnalyze)
    - PipelineCompilationJob::PrepareJob =>
      - PipelineImpl::CreateGraph =>
        - GraphBuilderPhase::Run (turbofan IR from bytecode array)
        - (and a lot other phases e.g. TyperPhase)
    - PipelineCompilationJob::ExecuteJob => PipelineImpl::OptimizeGraph (e.g. LoopPeelingPhase)
    - PipelineCompilationJob::FinalizeJob =>
      - PipelineImpl::GenerateCode (I roughly followed this path above)
```


- Q. builtin: Interpreter entry trampoline

```
- Builtins::Generate_InterpreterEntryTrampoline (x64/builtins-x64.cc) => ??
```


# Some Examples

```
[ AST example ]
$ out.gn/x64.debug/v8_shell --print_ast
> (function fib(n) { return (n <= 1 ? n : fib(n - 1) + fib(n - 2)); })(10)
[generating interpreter code for user-defined function: fib]
--- AST ---
FUNC at 1
. KIND 0
. SUSPEND COUNT 0
. NAME "fib"
. INFERRED NAME ""
. PARAMS
. . VAR (mode = VAR) "n"
. EXPRESSION STATEMENT at -1
. . INIT at -1
. . . VAR PROXY local[0] (mode = CONST) "fib"
. . . THIS-FUNCTION at 1
. RETURN at 19
. . CONDITIONAL at 27
. . . CONDITION at 29
. . . . LTE at 29
. . . . . VAR PROXY parameter[0] (mode = VAR) "n"
. . . . . LITERAL 1
. . . THEN at 36
. . . . VAR PROXY parameter[0] (mode = VAR) "n"
. . . ELSE at 51
. . . . ADD at 51
. . . . . CALL Slot(1)
. . . . . . VAR PROXY local[0] (mode = CONST) "fib"
. . . . . . SUB at 46
. . . . . . . VAR PROXY parameter[0] (mode = VAR) "n"
. . . . . . . LITERAL 1
. . . . . CALL Slot(4)
. . . . . . VAR PROXY local[0] (mode = CONST) "fib"
. . . . . . SUB at 59
. . . . . . . VAR PROXY parameter[0] (mode = VAR) "n"
. . . . . . . LITERAL 2


[ Bytecode generation example ]
$ out.gn/x64.debug/v8_shell --print_bytecode
> (function fib(n) { return (n <= 1 ? n : fib(n - 1) + fib(n - 2)); })(10)
[generating bytecode for function: fib]
Parameter count 2
Frame size 40
   13 E> 0x171d3046f5e6 @    0 : 88                StackCheck
         0x171d3046f5e7 @    1 : 20 fe fa          Mov <closure>, r0
   19 S> 0x171d3046f5ea @    4 : 03 01             LdaSmi [1]
   29 E> 0x171d3046f5ec @    6 : 59 02 02          TestLessThanOrEqual a0, [2]
         0x171d3046f5ef @    9 : 7f 06             JumpIfFalse [6] (0x171d3046f5f5 @ 15)
         0x171d3046f5f1 @   11 : 1e 02             Ldar a0
         0x171d3046f5f3 @   13 : 72 23             Jump [35] (0x171d3046f616 @ 48)
         0x171d3046f5f5 @   15 : 04                LdaUndefined
         0x171d3046f5f6 @   16 : 1f f8             Star r2
   46 E> 0x171d3046f5f8 @   18 : 38 01 02 05       SubSmi [1], a0, [5]
         0x171d3046f5fc @   22 : 1f f7             Star r3
   40 E> 0x171d3046f5fe @   24 : 47 fa f8 f7 03    Call1 r0, r2, r3, [3]
         0x171d3046f603 @   29 : 1f f9             Star r1
         0x171d3046f605 @   31 : 04                LdaUndefined
         0x171d3046f606 @   32 : 1f f7             Star r3
   59 E> 0x171d3046f608 @   34 : 38 02 02 08       SubSmi [2], a0, [8]
         0x171d3046f60c @   38 : 1f f6             Star r4
   53 E> 0x171d3046f60e @   40 : 47 fa f7 f6 06    Call1 r0, r3, r4, [6]
   51 E> 0x171d3046f613 @   45 : 2c f9 09          Add r1, [9]
   66 S> 0x171d3046f616 @   48 : 8c                Return

- Notes (read BytecodeArray::Disassemble and BytecodeDecoder::Decode)
  - "S>" is statement and "E>" is expression (with source script offset on its left)
  - types of operands
    - "r" means register in register file (e.g. temporary value)
    - "a" represents argument
    - "[]" is for Uimm (unsigned immidiate) and some others
```

# TODO

- promise implementation
- debugger architecture
- phi node resolution
- de-optimization path
- OSR
- compiler testing (e.g. register allocation, instruction selection ?)
- MessageLoop
  - v8::platform::PumpMessageLoop ?
- (internal) Threads use
- GC
- Some Javascript feature I don't know
  - label jump: http://www.ecma-international.org/ecma-262/7.0/index.html#sec-labelled-statements
  - execution context: http://www.ecma-international.org/ecma-262/7.0/index.html#sec-executable-code-and-execution-contexts


# References

- Spec: http://www.ecma-international.org/ecma-262/7.0/index.htm
- [Ignition Design Doc](https://docs.google.com/document/d/11T2CRex9hXxoJwbYqVQ32yIPMh0uouUZLdyrtmMoL44/edit?ts=56f27d9d#heading=h.6jz9dj3bnr8t)
- [Turbofan](https://github.com/v8/v8/wiki/TurboFan)

  - [Turbofan IR](https://docs.google.com/presentation/d/1Z9iIHojKDrXvZ27gRX51UxHD-bKf1QcPzSijntpMJBM/edit#slide=id.p)
  - [An overview of the TurboFan compiler](https://docs.google.com/presentation/d/1H1lLsbclvzyOF3IUR05ZUaZcqDxo7_-8f4yJoxdMooU/edit#slide=id.p)
  - [Turbofan JIT design](https://docs.google.com/presentation/d/1sOEF4MlF7LeO7uq-uThJSulJlTh--wgLeaVibsbb3tc/edit#slide=id.p)

- OSR: http://stackoverflow.com/questions/9105505/differences-between-just-in-time-compilation-and-on-stack-replacement
