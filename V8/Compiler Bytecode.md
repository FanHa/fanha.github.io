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
  BuildAwait(expr->position());
  // 
  BuildIncrementBlockCoverageCounterIfEnabled(expr,
                                              SourceRangeKind::kContinuation);
}

//TODO BuildAwait
```


##### Call

##### ThisExpression
