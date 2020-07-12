## Bytecode 
V8在parse阶段先将源码解析成了AST之后,需要再进一步将AST的编译成更靠近底层汇编语言的Bytecode

--- 
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
---
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
---
#### VisitThisFunctionVariable
js 的‘this’也是一个Variable实例
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitThisFunctionVariable(Variable* variable) {
  if (variable == nullptr) return;

  // ‘Register::function_closure()’用来新建一个指向functionClosure的Register,
  // 然后生成一个把这个Register的值加载到AccumulateRegister的Bytecode
  builder()->LoadAccumulatorWithRegister(Register::function_closure());
  // 生成 将当前AccumulateRegister的内容(即functionClosure) 移动(写入)variable所指向的位置
  BuildVariableAssignment(variable, Token::INIT, HoleCheckMode::kElided);
}
```
```cpp
void BytecodeGenerator::BuildVariableAssignment(
    Variable* variable, Token::Value op, HoleCheckMode hole_check_mode,
    LookupHoistingMode lookup_hoisting_mode) {
  VariableMode mode = variable->mode();
  RegisterAllocationScope assignment_register_scope(this);
  BytecodeLabel end_label;
  switch (variable->location()) {
    case VariableLocation::PARAMETER:
    case VariableLocation::LOCAL: {
      // 创建一个指向变量地址的Register
      Register destination;
      if (VariableLocation::PARAMETER == variable->location()) {
        if (variable->IsReceiver()) {
          destination = builder()->Receiver();
        } else {
          destination = builder()->Parameter(variable->index());
        }
      } else {
        destination = builder()->Local(variable->index());
      }

      // ...
      if (mode != VariableMode::kConst || op == Token::INIT) {
        // 将AccumulateRegister里的值写入前面的Register指向的位置,
        builder()->StoreAccumulatorInRegister(destination);
      } else if (variable->throw_on_const_assignment(language_mode())) {
        // ...
      }
      break;
    }
  }
}
```
---
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
      // 判断变量是否需要初始化一个Hole占位值
      if (variable->binding_needs_init()) {
        // 生成初始化变量的Bytecode指令;
        // variable->index() 即变量的位置,首先初始化一个以变量位置为目的地的Register;
        Register destination(builder()->Local(variable->index()));
        // 调用LoadTheHole给 accumulateRegister设置一个“theHole”值(占位),
        // 然后调用StoreAccumulatorInRegister把这个“theHole”值写到前面生成的Register指向的位置,
        // 完成变量的初始化
        builder()->LoadTheHole().StoreAccumulatorInRegister(destination);
      }
      break;
    // ...
  }
}
```

##### VisitFunctionDeclaration
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitFunctionDeclaration(FunctionDeclaration* decl) {
  Variable* variable = decl->var();
  if (!variable->is_used()) return;

  switch (variable->location()) {
    case VariableLocation::UNALLOCATED:
      UNREACHABLE();
    case VariableLocation::PARAMETER:
    case VariableLocation::LOCAL: {
      // 先为function的内容创建Bytecode
      VisitFunctionLiteral(decl->fun());
      // 为function变量名创建Bytecode,这一步会把上一步生成的Bytecode写入变量所在的Register位置
      BuildVariableAssignment(variable, Token::INIT, HoleCheckMode::kElided);
      break;
    }
    // ...
  }

}

void BytecodeGenerator::VisitFunctionLiteral(FunctionLiteral* expr) {
  uint8_t flags = CreateClosureFlags::Encode(
      expr->pretenure(), closure_scope()->is_function_scope(),
      info()->might_always_opt());
  // 初始化一个Bytecode 入口entry
  size_t entry = builder()->AllocateDeferredConstantPoolEntry();
  // 调用CreateClosure创建函数的Bytecode,并绑定入口entry
  builder()->CreateClosure(entry, GetCachedCreateClosureSlot(expr), flags);
  // 将function的expression与入口配个队push到function_literals里,
  // 猜想是为了后面遇到该function表达式式可以找到Bytecode的入口
  function_literals_.push_back(std::make_pair(expr, entry));
  // 如果function标记了Eager选项,则需要把function里面的内容
  AddToEagerLiteralsIfEager(expr);
}
```

---
#### VisitStatements
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitStatements(
    const ZonePtrList<Statement>* statements) {
  for (int i = 0; i < statements->length(); i++) {
    // 遍历所有statements, Visit每一个stmt生成Bytecode
    RegisterAllocationScope allocation_scope(this);
    Statement* stmt = statements->at(i);
    Visit(stmt);
    // 遇到了需要退出的情况(某些语法?)则无需继续生成Bytecode
    if (builder()->RemainderOfBlockIsDead()) break;
  }
}
```
以下是Statement的类型,大多是js语法保留的关键字,可以生成固定的Bytecode;
ExpressionStatement是个例外,比如前面声明并定义了function func();
然后调用func(),这个调用就是一个ExpressionStatement;
```cpp
// src/ast/ast.h
#define ITERATION_NODE_LIST(V) \
  V(DoWhileStatement)          \
  V(WhileStatement)            \
  V(ForStatement)              \
  V(ForInStatement)            \
  V(ForOfStatement)

#define BREAKABLE_NODE_LIST(V) \
  V(Block)                     \
  V(SwitchStatement)

#define STATEMENT_NODE_LIST(V)    \
  ITERATION_NODE_LIST(V)          \
  BREAKABLE_NODE_LIST(V)          \
  V(ExpressionStatement)          \
  V(EmptyStatement)               \
  V(SloppyBlockFunctionStatement) \
  V(IfStatement)                  \
  V(ContinueStatement)            \
  V(BreakStatement)               \
  V(ReturnStatement)              \
  V(WithStatement)                \
  V(TryCatchStatement)            \
  V(TryFinallyStatement)          \
  V(DebuggerStatement)            \
  V(InitializeClassMembersStatement)
```

##### VisitExpressionStatement
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitExpressionStatement(ExpressionStatement* stmt) {
  // 保存Statement的Position信息;
  // ?这个似乎不是生成Bytecode,只是保存信息,供后面其他情况使用
  builder()->SetStatementPosition(stmt);
  // 生成Bytecode
  VisitForEffect(stmt->expression());
}

void BytecodeGenerator::VisitForEffect(Expression* expr) {
  EffectResultScope effect_scope(this);
  Visit(expr);
}
```
ExpressionStatement列表
```cpp
#define LITERAL_NODE_LIST(V) \
  V(RegExpLiteral)           \
  V(ObjectLiteral)           \
  V(ArrayLiteral)

#define EXPRESSION_NODE_LIST(V) \
  LITERAL_NODE_LIST(V)          \
  V(Assignment)                 \
  V(Await)                      \
  V(BinaryOperation)            \
  V(NaryOperation)              \
  V(Call)                       \
  V(CallNew)                    \
  V(CallRuntime)                \
  V(ClassLiteral)               \
  V(CompareOperation)           \
  V(CompoundAssignment)         \
  V(Conditional)                \
  V(CountOperation)             \
  V(DoExpression)               \
  V(EmptyParentheses)           \
  V(FunctionLiteral)            \
  V(GetTemplateObject)          \
  V(ImportCallExpression)       \
  V(Literal)                    \
  V(NativeFunctionLiteral)      \
  V(OptionalChain)              \
  V(Property)                   \
  V(Spread)                     \
  V(StoreInArrayLiteral)        \
  V(SuperCallReference)         \
  V(SuperPropertyReference)     \
  V(TemplateLiteral)            \
  V(ThisExpression)             \
  V(Throw)                      \
  V(UnaryOperation)             \
  V(VariableProxy)              \
  V(Yield)                      \
  V(YieldStar)
```

##### VisitAssignment
给变量赋值时的Bytecode
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitAssignment(Assignment* expr) {
  // 找到赋值语句的左值所在的地址
  AssignmentLhsData lhs_data = PrepareAssignmentLhs(expr->target());
  // 生成将右值写入accumulateRegister的Bytecode
  VisitForAccumulatorValue(expr->value());
  // 记录赋值语句的代码位置信息 ??TODO 这个信息用来干嘛?
  builder()->SetExpressionPosition(expr);
  // 生成将AccumulateRegister里的值(即右值)写入左值的Bytecode
  BuildAssignment(lhs_data, expr->op(), expr->lookup_hoisting_mode());
}
```

##### VisitAwait
await语句的Bytecode
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitAwait(Await* expr) {
  // 记录await语句的代码位置信息
  builder()->SetExpressionPosition(expr);
  // 将await后面语句作为整体写入AccumulateRegister的Bytecode
  VisitForAccumulatorValue(expr->expression());
  // 生成Await逻辑的Bytecode
  // #ToExpand
  BuildAwait(expr->position());
  // 
  BuildIncrementBlockCoverageCounterIfEnabled(expr,
                                              SourceRangeKind::kContinuation);
}

void BytecodeGenerator::BuildAwait(int position) {
  {
    // Await(operand) and suspend.
    RegisterAllocationScope register_scope(this);

    Runtime::FunctionId await_intrinsic_id;
    if (IsAsyncGeneratorFunction(function_kind())) {
      // ...
    } else {
      await_intrinsic_id = catch_prediction() == HandlerTable::ASYNC_AWAIT
                               ? Runtime::kInlineAsyncFunctionAwaitUncaught
                               : Runtime::kInlineAsyncFunctionAwaitCaught;
    }
    // 新建两个Register;??TODO作用
    RegisterList args = register_allocator()->NewRegisterList(2);
    builder()
        // 第一个Register用来放将来会到的结果
        ->MoveRegister(generator_object(), args[0])
        // 然后把当前Accumulate里的值(已经在上一层写入了await后面的具体内容)写入第二个Register
        .StoreAccumulatorInRegister(args[1])
        // 生成CallRuntime的Bytecode; TODO CallRuntime 和 Call有什么区别
        .CallRuntime(await_intrinsic_id, args);
  }
  // 保存代码流挂起的位置信息,(await需要等待await后面表达式的结果完成了才会继续执行后面的代码)
  // #ToExpand
  BuildSuspendPoint(position);

  // TODO

  Register input = register_allocator()->NewRegister();
  Register resume_mode = register_allocator()->NewRegister();

  // Now dispatch on resume mode.
  BytecodeLabel resume_next;
  builder()
      ->StoreAccumulatorInRegister(input)
      .CallRuntime(Runtime::kInlineGeneratorGetResumeMode, generator_object())
      .StoreAccumulatorInRegister(resume_mode)
      .LoadLiteral(Smi::FromInt(JSGeneratorObject::kNext))
      .CompareReference(resume_mode)
      .JumpIfTrue(ToBooleanMode::kAlreadyBoolean, &resume_next);

  // Resume with "throw" completion (rethrow the received value).
  // TODO(leszeks): Add a debug-only check that the accumulator is
  // JSGeneratorObject::kThrow.
  builder()->LoadAccumulatorWithRegister(input).ReThrow();

  // Resume with next.
  builder()->Bind(&resume_next);
  builder()->LoadAccumulatorWithRegister(input);
}

// TODO
void BytecodeGenerator::BuildSuspendPoint(int position) {
  // Because we eliminate jump targets in dead code, we also eliminate resumes
  // when the suspend is not emitted because otherwise the below call to Bind
  // would start a new basic block and the code would be considered alive.
  if (builder()->RemainderOfBlockIsDead()) {
    return;
  }
  const int suspend_id = suspend_count_++;

  RegisterList registers = register_allocator()->AllLiveRegisters();

  // Save context, registers, and state. This bytecode then returns the value
  // in the accumulator.
  builder()->SetExpressionPosition(position);
  builder()->SuspendGenerator(generator_object(), registers, suspend_id);

  // Upon resume, we continue here.
  builder()->Bind(generator_jump_table_, suspend_id);

  // Clobbers all registers and sets the accumulator to the
  // [[input_or_debug_pos]] slot of the generator object.
  builder()->ResumeGenerator(generator_object(), registers);
}
```


##### Call
函数 或 方法的Bytecode
```cpp
void BytecodeGenerator::VisitCall(Call* expr) {
  Expression* callee_expr = expr->expression();
  Call::CallType call_type = expr->GetCallType();

  // ...
  // 为函数本身 和 函数的参数们各自新建Register
  Register callee = register_allocator()->NewRegister();
  RegisterList args = register_allocator()->NewGrowableRegisterList();

  switch (call_type) {
    case Call::NAMED_PROPERTY_CALL:
    case Call::KEYED_PROPERTY_CALL:
    case Call::PRIVATE_CALL: {
      Property* property = callee_expr->AsProperty();
      // 因为函数可以调用自己,所以需要将函数自己(callee)作为一个属性加入到函数的参数列表中(即在参数Register列表里加一个存放callee的Register)
      VisitAndPushIntoRegisterList(property->obj(), &args);
      VisitPropertyLoadForRegister(args.last_register(), property, callee);
      break;
    }
    // ...
  }

  // 生成将函数参数载入到前面新建的Register列表的Bytecode
  VisitArguments(expr->arguments(), &args);
  // ...
  // 保存语句的位置信息
  builder()->SetExpressionPosition(expr);
  // 为不同类型的Call生成Bytecode
  if (is_spread_call) {
    // ...
  } else if (optimize_as_one_shot) {
    // ...
  } else if (call_type == Call::NAMED_PROPERTY_CALL ||
             call_type == Call::KEYED_PROPERTY_CALL) {
    DCHECK(!implicit_undefined_receiver);
    builder()->CallProperty(callee, args,
                            feedback_index(feedback_spec()->AddCallICSlot()));
  } else if (implicit_undefined_receiver) {
    builder()->CallUndefinedReceiver(
        callee, args, feedback_index(feedback_spec()->AddCallICSlot()));
  } else {
    builder()->CallAnyReceiver(
        callee, args, feedback_index(feedback_spec()->AddCallICSlot()));
  }
}
```

##### ThisExpression
这个应该是显式的‘this’值的Bytecode生成
```cpp
// src/interpreter/bytecode-generator.cc
void BytecodeGenerator::VisitThisExpression(ThisExpression* expr) {
  BuildThisVariableLoad();
}

void BytecodeGenerator::BuildThisVariableLoad() {
  // 向外找到可以作为ReceiverScope的Scope
  DeclarationScope* receiver_scope = closure_scope()->GetReceiverScope();
  // 为receiver(即this指向的地方)新建一个var
  Variable* var = receiver_scope->receiver();
  // 
  HoleCheckMode hole_check_mode =
      IsDerivedConstructor(receiver_scope->function_kind())
          ? HoleCheckMode::kRequired
          : HoleCheckMode::kElided;
  // 生成加载代表receiver的var的Bytecode
  BuildVariableLoad(var, hole_check_mode);
}
```
