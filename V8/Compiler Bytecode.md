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
    // 如果节点已经被编译过了,则直接跳过
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
  //...
    GenerateBytecodeBody();
  //...
}

void BytecodeGenerator::GenerateBytecodeBody() {
  // 如果当前要生成的内容是包裹在一个function里面的,则需要生成function的参数的Bytecode
  // 注:AST最外层的节点没有这些
  VisitArgumentsObject(closure_scope()->arguments());
  Variable* rest_parameter = closure_scope()->rest_parameter();
  VisitRestArgumentsArray(rest_parameter);

  // 生成JS语法里的显示的,隐式的“this”的Bytecode,
  // 即当遇到一个变量,当前AST节点的Scope找不到,需要通过Bytecode里的信息跳转到哪一块代码来搜索这个变量
  VisitThisFunctionVariable(closure_scope()->function_var());
  VisitThisFunctionVariable(closure_scope()->this_function_var());

  // Build assignment to {new.target} variable if it is used.
  VisitNewTargetVariable(closure_scope()->new_target_var());

  //...

  // 当前AST节点的Scope的变量声明的Bytecode
  if (closure_scope()->is_script_scope()) {
    VisitGlobalDeclarations(closure_scope()->declarations());
  } else {
    VisitDeclarations(closure_scope()->declarations());
  }

  // 当前AST节点的引入的模块的Bytecode生成
  VisitModuleNamespaceImports();

  // 与Class相关的AST节点的Bytecode
  if (IsBaseConstructor(function_kind())) {
    if (literal->requires_brand_initialization()) {
      BuildPrivateBrandInitialization(builder()->Receiver());
    }

    if (literal->requires_instance_members_initializer()) {
      BuildInstanceMemberInitialization(Register::function_closure(),
                                        builder()->Receiver());
    }
  }

  // 当前AST节点的Statement语句的Bytecode生成
  VisitStatements(literal->body());

  // ...
}

```

#### VisitThisFunctionVariable

#### VisitDeclarations
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitDeclarations(Declaration::List* declarations) {
  for (Declaration* decl : *declarations) {
    RegisterAllocationScope register_scope(this);
    // Visit() 是个宏,根据decl的类型决定调用的方法;
    // JS目前有两种Declaration:
    // FunctionDeclaration 和  VariableDeclaration;
    // 分别调用的是VisitFunctionDeclaration 和VisitVariableDeclaration
    // TODO Class 声明是在哪里?
    Visit(decl);
  }
}
```

##### VisitVariableDeclaration
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitVariableDeclaration(VariableDeclaration* decl) {
  Variable* variable = decl->var();
  // Unused variables don't need to be visited.
  if (!variable->is_used()) return;

  switch (variable->location()) {
    case VariableLocation::UNALLOCATED:
      UNREACHABLE();
    case VariableLocation::LOCAL:
      if (variable->binding_needs_init()) {
        Register destination(builder()->Local(variable->index()));
        builder()->LoadTheHole().StoreAccumulatorInRegister(destination);
      }
      break;
    case VariableLocation::PARAMETER:
      if (variable->binding_needs_init()) {
        Register destination(builder()->Parameter(variable->index()));
        builder()->LoadTheHole().StoreAccumulatorInRegister(destination);
      }
      break;
    case VariableLocation::REPL_GLOBAL:
      // REPL let's are stored in script contexts. They get initialized
      // with the hole the same way as normal context allocated variables.
    case VariableLocation::CONTEXT:
      if (variable->binding_needs_init()) {
        DCHECK_EQ(0, execution_context()->ContextChainDepth(variable->scope()));
        builder()->LoadTheHole().StoreContextSlot(execution_context()->reg(),
                                                  variable->index(), 0);
      }
      break;
    case VariableLocation::LOOKUP: {
      DCHECK_EQ(VariableMode::kDynamic, variable->mode());
      DCHECK(!variable->binding_needs_init());

      Register name = register_allocator()->NewRegister();

      builder()
          ->LoadLiteral(variable->raw_name())
          .StoreAccumulatorInRegister(name)
          .CallRuntime(Runtime::kDeclareEvalVar, name);
      break;
    }
    case VariableLocation::MODULE:
      if (variable->IsExport() && variable->binding_needs_init()) {
        builder()->LoadTheHole();
        BuildVariableAssignment(variable, Token::INIT, HoleCheckMode::kElided);
      }
      // Nothing to do for imports.
      break;
  }
}
```

##### VisitFunctionDeclaration
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitFunctionDeclaration(FunctionDeclaration* decl) {
  Variable* variable = decl->var();
  DCHECK(variable->mode() == VariableMode::kLet ||
         variable->mode() == VariableMode::kVar ||
         variable->mode() == VariableMode::kDynamic);
  // Unused variables don't need to be visited.
  if (!variable->is_used()) return;

  switch (variable->location()) {
    case VariableLocation::UNALLOCATED:
      UNREACHABLE();
    case VariableLocation::PARAMETER:
    case VariableLocation::LOCAL: {
      VisitFunctionLiteral(decl->fun());
      BuildVariableAssignment(variable, Token::INIT, HoleCheckMode::kElided);
      break;
    }
    case VariableLocation::REPL_GLOBAL:
    case VariableLocation::CONTEXT: {
      DCHECK_EQ(0, execution_context()->ContextChainDepth(variable->scope()));
      VisitFunctionLiteral(decl->fun());
      builder()->StoreContextSlot(execution_context()->reg(), variable->index(),
                                  0);
      break;
    }
    case VariableLocation::LOOKUP: {
      RegisterList args = register_allocator()->NewRegisterList(2);
      builder()
          ->LoadLiteral(variable->raw_name())
          .StoreAccumulatorInRegister(args[0]);
      VisitFunctionLiteral(decl->fun());
      builder()->StoreAccumulatorInRegister(args[1]).CallRuntime(
          Runtime::kDeclareEvalFunction, args);
      break;
    }
    case VariableLocation::MODULE:
      DCHECK_EQ(variable->mode(), VariableMode::kLet);
      DCHECK(variable->IsExport());
      VisitForAccumulatorValue(decl->fun());
      BuildVariableAssignment(variable, Token::INIT, HoleCheckMode::kElided);
      break;
  }
  DCHECK_IMPLIES(
      eager_inner_literals_ != nullptr && decl->fun()->ShouldEagerCompile(),
      IsInEagerLiterals(decl->fun(), *eager_inner_literals_));
}

```


#### VisitStatements

