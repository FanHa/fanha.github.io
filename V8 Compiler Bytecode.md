## Bytecode
V8在parse阶段先将源码解析成了AST之后,需要再进一步将AST的编译成更靠近底层汇编语言的Bytecode

## 入口
V8 在CompileTopLevel时先解析了最外层的信息,并将解析结果存在了parse_info里
```cpp
// src/codegen/compiler.cc
MaybeHandle<SharedFunctionInfo> CompileToplevel(
    ParseInfo* parse_info, Handle<Script> script, Isolate* isolate,
    IsCompiledScope* is_compiled_scope) {
  // ...
  // 将最外层代码解析成ast,
  // 注: js解析成的AST的最外层节点类型就是“Program”
  if (parse_info->literal() == nullptr &&
      !parsing::ParseProgram(parse_info, script, isolate)) {
    return MaybeHandle<SharedFunctionInfo>();
  }
  // ...
  // GenerateUnoptimizedCodeForToplevel就是将前面生成的最外层AST转变成更靠近汇编语言的Bytecode
  MaybeHandle<SharedFunctionInfo> shared_info =
      GenerateUnoptimizedCodeForToplevel(
          isolate, script, parse_info, isolate->allocator(), is_compiled_scope);
  // ...
  // TODO
  FinalizeScriptCompilation(isolate, script, parse_info);
  return shared_info;
}

MaybeHandle<SharedFunctionInfo> GenerateUnoptimizedCodeForToplevel(
    Isolate* isolate, Handle<Script> script, ParseInfo* parse_info,
    AccountingAllocator* allocator, IsCompiledScope* is_compiled_scope) {
  // ...
  // 为最外层的AST 的literal新建一个Handle(未编译)
  Handle<SharedFunctionInfo> top_level =
      isolate->factory()->NewSharedFunctionInfoForLiteral(parse_info->literal(),
                                                          script, true);
  // 初始化一个vector functions_to_compile,并将AST最外层节点的解析信息push到这个vector里,
  // 通过一个while循环不断将function_to_compile 里的FunctionLiteral转变成Bytecode,
  std::vector<FunctionLiteral*> functions_to_compile;
  functions_to_compile.push_back(parse_info->literal());

  while (!functions_to_compile.empty()) {
    // 取出当前要编译的任务
    FunctionLiteral* literal = functions_to_compile.back();
    functions_to_compile.pop_back();
    // 取出前面根据literal已经建好的handle
    Handle<SharedFunctionInfo> shared_info =
        Compiler::GetSharedFunctionInfo(literal, script, isolate);
    // 因为还未开始编译所以这里不能continue
    if (shared_info->is_compiled()) continue;
    // ...

    // 新建一个“InterpreterCompilationJob(继承自UnoptimizedCompilationJob)”实例;
    // 这里把“&functions_to_compile”当成参数传入了实例,因为在某些贪婪算法下,编译的同时又会将新的需要编译的函数literal push到这个functions_to_compile中,
    // 那么下一个while循环时判断不为空,则会继续编译
    std::unique_ptr<UnoptimizedCompilationJob> job(
        interpreter::Interpreter::NewCompilationJob(
            parse_info, literal, allocator, &functions_to_compile));

    // 执行编译
    if (job->ExecuteJob() == CompilationJob::FAILED ||
        FinalizeUnoptimizedCompilationJob(job.get(), shared_info, isolate) ==
            CompilationJob::FAILED) {
      return MaybeHandle<SharedFunctionInfo>();
    }

    // ...
  }

  // Character stream shouldn't be used again.
  parse_info->ResetCharacterStream();

  return top_level;
}
```

### ExecuteJob
```cpp
// src/codegen/compiler.cc
CompilationJob::Status UnoptimizedCompilationJob::ExecuteJob() {
  DisallowHeapAccess no_heap_access;
  // ...
  return UpdateState(ExecuteJobImpl(), State::kReadyToFinalize);
}
```

```cpp
// src/interpreter/interpreter.cc
InterpreterCompilationJob::Status InterpreterCompilationJob::ExecuteJobImpl() {
  // 调用内部generator的GenerateBytecode方法生成Bytecode;
  // 注: generotor() 返回一个 “BytecodeGenerator”实例
  generator()->GenerateBytecode(stack_limit());

  if (generator()->HasStackOverflow()) {
    return FAILED;
  }
  return SUCCEEDED;
}
```

```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::GenerateBytecode(uintptr_t stack_limit) {
  DisallowHeapAllocation no_allocation;
  DisallowHandleAllocation no_handles;
  DisallowHandleDereference no_deref;

  InitializeAstVisitor(stack_limit);

  // Initialize the incoming context.
  ContextScope incoming_context(this, closure_scope());

  // Initialize control scope.
  ControlScopeForTopLevel control(this);

  RegisterAllocationScope register_scope(this);

  AllocateTopLevelRegisters();

  // Perform a stack-check before the body.
  builder()->StackCheck(info()->literal()->start_position());

  if (info()->literal()->CanSuspend()) {
    BuildGeneratorPrologue();
  }

  if (closure_scope()->NeedsContext() && !closure_scope()->is_script_scope()) {
    // Push a new inner context scope for the function.
    BuildNewLocalActivationContext();
    ContextScope local_function_context(this, closure_scope());
    BuildLocalActivationContextInitialization();
    GenerateBytecodeBody();
  } else {
    GenerateBytecodeBody();
  }

  // Check that we are not falling off the end.
  DCHECK(builder()->RemainderOfBlockIsDead());
}
```

