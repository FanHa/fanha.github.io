### 声明函数方式
```js
//  普通函数声明
function funcA (paramA, paramB) {
    // ...
}
```

### 解析function入口
```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseStatementListItem() {
  switch (peek()) {
    case Token::FUNCTION:
      // 解析语句遇到‘function’ Token时,进入HoistableSeclaration逻辑;
      // Hoistable表明function声明是变量提升,即function在同一个上下文中声明可以落后于使用
      return ParseHoistableDeclaration(nullptr, false);
    // ...
  }
}

// 封装ParseHoistableDeclaration,消费掉‘function’Token,初始化一些变量并调用重载版的ParseHoistableDeclaration
template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseHoistableDeclaration(
    ZonePtrList<const AstRawString>* names, bool default_export) {
  Consume(Token::FUNCTION);

  int pos = position();
  ParseFunctionFlags flags = ParseFunctionFlag::kIsNormal;
  if (Check(Token::MUL)) {
    flags |= ParseFunctionFlag::kIsGenerator;
  }
  return ParseHoistableDeclaration(pos, flags, names, default_export);
}


template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseHoistableDeclaration(
    int pos, ParseFunctionFlags flags, ZonePtrList<const AstRawString>* names,
    bool default_export) {
  // ...

  IdentifierT name;
  FunctionNameValidity name_validity;
  IdentifierT variable_name;
  if (peek() == Token::LPAREN) {
    // ‘function’Token后面直接接‘(‘的情形只有在export时才有效
    if (default_export) {
      impl()->GetDefaultStrings(&name, &variable_name);
      name_validity = kSkipFunctionNameCheck;
    } else {
      ReportMessage(MessageTemplate::kMissingFunctionName);
      return impl()->NullStatement();
    }
  } else {
    // 一般形式的 function aaa(){}

    // 判断函数名是否是系统保留名称
    bool is_strict_reserved = Token::IsStrictReservedWord(peek());
    // 获取函数名
    name = ParseIdentifier();
    name_validity = is_strict_reserved ? kFunctionNameIsStrictReserved
                                       : kFunctionNameValidityUnknown;
    variable_name = name;
  }

  FuncNameInferrerState fni_state(&fni_);
  impl()->PushEnclosingName(name);

  // 获取函数类型, 因为*generator函数, async函数, function里的方法(method)都是走这一解析流程;
  // 需要在这里做一个标记便于后面走不同的解析流程
  FunctionKind function_kind = FunctionKindFor(flags);

  // 这里就是对function内容的解析;
  // 这个ParseFunctionLiteral解析是一层封装,v8对函数的解析分为 parse(贪婪) 和 preparse(保守);
  // parse解析的更彻底;
  // preparse解析得相对肤浅,只解析必要信息,只有在后面真正使用这个函数时才会激活真正的parse,类似于“懒加载”
  FunctionLiteralT function = impl()->ParseFunctionLiteral(
      name, scanner()->location(), name_validity, function_kind, pos,
      FunctionSyntaxKind::kDeclaration, language_mode(), nullptr);

  // ...
  // 将函数名与函数内容绑定作具体声明
  return impl()->DeclareFunction(variable_name, function, mode, kind, pos,
                                 end_position(), names);
}
```

#### ParseFunctionLiteral 的贪心与延迟
```cpp
// src/parsing/parser.cc
FunctionLiteral* Parser::ParseFunctionLiteral(
    const AstRawString* function_name, Scanner::Location function_name_location,
    FunctionNameValidity function_name_validity, FunctionKind kind,
    int function_token_pos, FunctionSyntaxKind function_syntax_kind,
    LanguageMode language_mode,
    ZonePtrList<const AstRawString>* arguments_for_wrapped_function) {
  // ...
  int pos = function_token_pos == kNoSourcePosition ? peek_position()
  // ...
  // 根据上下文设置贪婪hint,后面需要用这个hint来决定是否贪婪还是延迟解析
  FunctionLiteral::EagerCompileHint eager_compile_hint =
      function_state_->next_function_is_likely_called() || is_wrapped
          ? FunctionLiteral::kShouldEagerCompile
          : default_eager_compile_hint();

  // 贪婪还是延迟解析需要一系列上下文共同决定..
  const bool is_lazy =
      eager_compile_hint == FunctionLiteral::kShouldLazyCompile;
  const bool is_top_level = AllowsLazyParsingWithoutUnresolvedVariables();
  const bool is_eager_top_level_function = !is_lazy && is_top_level;
  const bool is_lazy_top_level_function = is_lazy && is_top_level;
  const bool is_lazy_inner_function = is_lazy && !is_top_level;

  // 根据前面的hint设置当前解析是否是需要延迟解析的inner函数;
  // 只有在开启了parseLazily 并且前面的 ‘is_lazy_inner_function’ 设置为true时,将‘should_preparse_inner’ 设置为true
  const bool should_preparse_inner = parse_lazily() && is_lazy_inner_function;

  // 现代js解析器已经超越了传统的“js是单线程运行”的禁锢观念,比如这里可以先只简单的preparse函数,然后把详细的解析任务分发给一个平行的线程
  bool should_post_parallel_task =
      parse_lazily() && is_eager_top_level_function &&
      FLAG_parallel_compile_tasks && info()->parallel_tasks() &&
      scanner()->stream()->can_be_cloned_for_parallel_access();

  // 根据前面的解析结果设置‘should_preparse‘值
  bool should_preparse = (parse_lazily() && is_lazy_top_level_function) ||
                         should_preparse_inner || should_post_parallel_task;

  // 初始化函数body里的基本信息;
  // 将function的body里的内容初始化为ScopedPtrList
  ScopedPtrList<Statement> body(pointer_buffer());
  int expected_property_count = 0;
  int suspend_count = -1;
  int num_parameters = -1;
  int function_length = -1;
  bool has_duplicate_parameters = false;
  int function_literal_id = GetNextFunctionLiteralId();
  ProducedPreparseData* produced_preparse_data = nullptr;

  // This Scope lives in the main zone. We'll migrate data into that zone later.
  Zone* parse_zone = should_preparse ? &preparser_zone_ : zone();
  // 为函数创建一个新的Scope, GC相关??
  DeclarationScope* scope = NewFunctionScope(kind, parse_zone);
  SetLanguageMode(scope, language_mode);

  scope->set_start_position(position());

  // 如果上面的分析决定应该使用延迟(preparse),则调用SkipFunction;
  // 但是这个SkipFunction依然需要参考当前的各个中泰条件,是不确定能否延迟解析成功,仍然会有返回false的情况
  bool did_preparse_successfully =
      should_preparse &&
      SkipFunction(function_name, kind, function_syntax_kind, scope,
                   &num_parameters, &function_length, &produced_preparse_data);

  if (!did_preparse_successfully) {
    // 实在不能延迟了,消费掉‘(’,开始动手解析(ParseFunction)
    if (should_preparse) Consume(Token::LPAREN);
    should_post_parallel_task = false;
    ParseFunction(&body, function_name, pos, kind, function_syntax_kind, scope,
                  &num_parameters, &function_length, &has_duplicate_parameters,
                  &expected_property_count, &suspend_count,
                  arguments_for_wrapped_function);
  }

  // ...

  // 以前面的解析结果初始化一个需要返回的FunctionLiteral 数据结构
  FunctionLiteral* function_literal = factory()->NewFunctionLiteral(
      function_name, scope, body, expected_property_count, num_parameters,
      function_length, duplicate_parameters, function_syntax_kind,
      eager_compile_hint, pos, true, function_literal_id,
      produced_preparse_data);
  function_literal->set_function_token_position(function_token_pos);
  function_literal->set_suspend_count(suspend_count);

  // ...
  if (should_post_parallel_task) {
    // 当前面决定采用平行解析(即当前只preparse,然后把具体parse分配给另一个线程去作)方案时,把任务入队
    info()->parallel_tasks()->Enqueue(info(), function_name, function_literal);
  }

  return function_literal;
}

```

#### 延迟(SkipFunction)
```cpp
bool Parser::SkipFunction(const AstRawString* function_name, FunctionKind kind,
                          FunctionSyntaxKind function_syntax_kind,
                          DeclarationScope* function_scope, int* num_parameters,
                          int* function_length,
                          ProducedPreparseData** produced_preparse_data) {
  FunctionState function_state(&function_state_, &scope_, function_scope);
  function_scope->set_zone(&preparser_zone_);

  if (consumed_preparse_data_) {
    // 似乎存在某种机制缓存preparse_data ??
    // 这里直接从consumed_preparse_data中取出要lazyParse的数据,做好处理,然后return true了
    int end_position;
    LanguageMode language_mode;
    int num_inner_functions;
    bool uses_super_property;
    if (stack_overflow()) return true;
    *produced_preparse_data =
        consumed_preparse_data_->GetDataForSkippableFunction(
            main_zone(), function_scope->start_position(), &end_position,
            num_parameters, function_length, &num_inner_functions,
            &uses_super_property, &language_mode);

    function_scope->outer_scope()->SetMustUsePreparseData();
    function_scope->set_is_skipped_function(true);
    function_scope->set_end_position(end_position);
    scanner()->SeekForward(end_position - 1);
    Expect(Token::RBRACE);
    SetLanguageMode(function_scope, language_mode);
    if (uses_super_property) {
      function_scope->RecordSuperPropertyUsage();
    }
    SkipFunctionLiterals(num_inner_functions);
    function_scope->ResetAfterPreparsing(ast_value_factory_, false);
    return true;
  }

  // ... 
  // 调用PreParser解析器的方法,正式预解析
  PreParser::PreParseResult result = reusable_preparser()->PreParseFunction(
      function_name, kind, function_syntax_kind, function_scope, use_counts_,
      produced_preparse_data, this->script_id());

  if (result == PreParser::kPreParseStackOverflow) {
    // ... 各种不能preParse的情况1
  } else if (pending_error_handler()->has_error_unidentifiable_by_preparser()) {
    // ... 各种不能preParse的情况2
    return false;
  } else if (pending_error_handler()->has_pending_error()) {
    // ... 各种不能preParse的情况3
  } else {
    // 这里应该是preParse解析成功后作的一些信息记录
    DCHECK(!pending_error_handler()->stack_overflow());
    set_allow_eval_cache(reusable_preparser()->allow_eval_cache());

    PreParserLogger* logger = reusable_preparser()->logger();
    function_scope->set_end_position(logger->end());
    Expect(Token::RBRACE);
    total_preparse_skipped_ +=
        function_scope->end_position() - function_scope->start_position();
    *num_parameters = logger->num_parameters();
    *function_length = logger->function_length();
    SkipFunctionLiterals(logger->num_inner_functions());
    if (!private_name_scope_iter.Done()) {
      private_name_scope_iter.GetScope()->MigrateUnresolvedPrivateNameTail(
          factory(), unresolved_private_tail);
    }
    function_scope->AnalyzePartially(this, factory(), MaybeParsingArrowhead());
  }

  return true;
}

```
##### PreParseFunction
```cpp
// src/parsing/preparser.cc
PreParser::PreParseResult PreParser::PreParseFunction(
    const AstRawString* function_name, FunctionKind kind,
    FunctionSyntaxKind function_syntax_kind, DeclarationScope* function_scope,
    int* use_counts, ProducedPreparseData** produced_preparse_data,
    int script_id) {
  // ...
  // preParse也需要初始化函数的参数
  PreParserFormalParameters formals(function_scope);

  // ...

  // 先新建一个独特的DataGatheringScope用来preparse
  PreparseDataBuilder::DataGatheringScope preparse_data_builder_scope(this);

  if (IsArrowFunction(kind)) {
    // 箭头函数的参数应该是已经在外面解析完成了
    formals.is_simple = function_scope->has_simple_parameters();
  } else {
    preparse_data_builder_scope.Start(function_scope);
    // 非箭头函数时需要新建一个Scope来解析函数参数
    ParameterDeclarationParsingScope formals_scope(this);
    // preparse 也需要真正的解析函数参数的形式,和检测各种可能的错误
    ParseFormalParameterList(&formals);
    if (formals_scope.has_duplicate()) formals.set_has_duplicate();
    if (!formals.is_simple) {
      BuildParameterInitializationBlock(formals);
    }
    // 参数包裹在括号里,需要一个‘)’表明参数解析完成
    Expect(Token::RPAREN);
    int formals_end_position = scanner()->location().end_pos;
    // ...
  }
  // ‘{'  来表明函数体的开始
  Expect(Token::LBRACE);
  // 将inner_scope指向function_scope
  DeclarationScope* inner_scope = function_scope;

  if (!formals.is_simple) {
    // 函数参数不是简单的参数时,新建一个scope存放参数
    inner_scope = NewVarblockScope();
    inner_scope->set_start_position(position());
  }

  {
    BlockState block_state(&scope_, inner_scope);
    // 这里有疑问,这个ParseStatementListAndLogFunction直接就解析了整个函数体里的内容;
    // 这里和普通的函数解析调用的代码是一个地方,调用ParseStatementList方法;
    // 但是在调用具体的ParseXXXLiteral时,如impl()->ParseFunctionLiteral;
    // 正常解析 impl()返回的是Parser,而这里返回的是PreParser,即在这里走向了不同的解析路线;
    // 这样看来preParse一个函数,依然会深入到该函数的每一个语句,取得该函数声明的一些变量,函数,类等等信息;
    // 但大部分语句不会再递归更深入地去解析了
    ParseStatementListAndLogFunction(&formals);
  }

  bool allow_duplicate_parameters = false;
  CheckConflictingVarDeclarations(inner_scope);

  // ...

  use_counts_ = nullptr;
  // 各种可能存在的错误处理
  if (stack_overflow()) {
    return kPreParseStackOverflow;
  } else if (pending_error_handler()->has_error_unidentifiable_by_preparser()) {
    return kPreParseNotIdentifiableError;
  } else if (has_error()) {
    DCHECK(pending_error_handler()->has_pending_error());
  } else {
    // 正常preParse
    if (!IsArrowFunction(kind)) {
      // 如果函数不是箭头函数,需要核查一下preParse出的变量的冲突,合规性等问题
      ValidateFormalParameters(language_mode(), formals,
                               allow_duplicate_parameters);
      //...

      // 尽管是preParse,外层的变量的声明还是要按部就班先占好声明位置的
      function_scope->DeclareArguments(ast_value_factory());

      DeclareFunctionNameVar(function_name, function_syntax_kind,
                             function_scope);

      if (preparse_data_builder_->HasData()) {
        *produced_preparse_data =
            ProducedPreparseData::For(preparse_data_builder_, main_zone());
      }
    }

    //...
  }

  //...
  return kPreParseSuccess;
}

```

#### 贪心(ParseFunction)
```cpp
// src/parsing/parser.cc
void Parser::ParseFunction(
    ScopedPtrList<Statement>* body, const AstRawString* function_name, int pos,
    FunctionKind kind, FunctionSyntaxKind function_syntax_kind,
    DeclarationScope* function_scope, int* num_parameters, int* function_length,
    bool* has_duplicate_parameters, int* expected_property_count,
    int* suspend_count,
    ZonePtrList<const AstRawString>* arguments_for_wrapped_function) {
  // 猜想延迟解析是个比较新的特性,可能有开关控制无论什么情况都不允许延迟解析
  ParsingModeScope mode(this, allow_lazy_ ? PARSE_LAZILY : PARSE_EAGERLY);

  FunctionState function_state(&function_state_, &scope_, function_scope);

  bool is_wrapped = function_syntax_kind == FunctionSyntaxKind::kWrapped;

  int expected_parameters_end_pos = parameters_end_pos_;

  // 初始化函数参数结构,为解析函数的参数作准备
  ParserFormalParameters formals(function_scope);

  {
    // 新建一块scope用来保存函数参数
    ParameterDeclarationParsingScope formals_scope(this);
    if (is_wrapped) {
      // ...
    } else {
      // 解析函数的参数
      ParseFormalParameterList(&formals);
      // ...
    }
    formals.duplicate_loc = formals_scope.duplicate_location();
  }

  *num_parameters = formals.num_parameters();
  *function_length = formals.function_length;

  AcceptINScope scope(this, true);
  // 解析函数内容
  ParseFunctionBody(body, function_name, pos, formals, kind,
                    function_syntax_kind, FunctionBodyType::kBlock);

  *has_duplicate_parameters = formals.has_duplicate();

  *expected_property_count = function_state.expected_property_count();
  *suspend_count = function_state.suspend_count();
}

```
##### 解析function参数(ParseFormalParameterList)
```cpp
// src/parsing/parser-base.h
template <typename Impl>
void ParserBase<Impl>::ParseFormalParameterList(FormalParametersT* parameters) {
  // 新建一块scope用来解析参数
  ParameterParsingScope scope(impl(), parameters);
  if (peek() != Token::RPAREN) {
    // 函数可能有多个参数,需要循环解析函数的每一个参数
    while (true) {
      // ...
      // 判断函数参数中是否有“...”形式
      parameters->has_rest = Check(Token::ELLIPSIS);
      // 解析参数, 这里应该调用应该只解析一个参数,对多个参数的解析是外面那层循环实现的
      ParseFormalParameter(parameters);

      if (parameters->has_rest) {
        // 函数参数是“...”形式时,标记函数参数的is_simple标记为false,并break参数解析流程
        parameters->is_simple = false;
        if (peek() == Token::COMMA) {
          impl()->ReportMessageAt(scanner()->peek_location(),
                                  MessageTemplate::kParamAfterRest);
          return;
        }
        break;
      }
      if (!Check(Token::COMMA)) break;
      if (peek() == Token::RPAREN) {
        // allow the trailing comma
        break;
      }
    }
  }
  // 将参数解析结果进行声明
  impl()->DeclareFormalParameters(parameters);
}


template <typename Impl>
void ParserBase<Impl>::ParseFormalParameter(FormalParametersT* parameters) {
  FuncNameInferrerState fni_state(&fni_);
  int pos = peek_position();
  auto declaration_it = scope()->declarations()->end();
  // 参数名
  ExpressionT pattern = ParseBindingPattern();
  if (impl()->IsIdentifier(pattern)) {
    ClassifyParameter(impl()->AsIdentifier(pattern), pos, end_position());
  } else {
    parameters->is_simple = false;
  }

  // 默认参数
  ExpressionT initializer = impl()->NullExpression();
  if (Check(Token::ASSIGN)) {
    // 参数名后接“=”时,参数有默认参数
    parameters->is_simple = false;

    if (parameters->has_rest) {
      ReportMessage(MessageTemplate::kRestDefaultInitializer);
      return;
    }

    AcceptINScope accept_in_scope(this, true);
    // 解析指定的默认参数形式
    initializer = ParseAssignmentExpression();
    impl()->SetFunctionNameFromIdentifierRef(initializer, pattern);
  }
  // ...
  // 正式把解析后的参数名和参数值加入到存放参数信息的区域
  impl()->AddFormalParameter(parameters, pattern, initializer, end_position(),
                             parameters->has_rest);
}

```

##### 解析functionBody(ParseFunctionBody)
```cpp
// src/parsing/parser-base.h
template <typename Impl>
void ParserBase<Impl>::ParseFunctionBody(
    StatementListT* body, IdentifierT function_name, int pos,
    const FormalParametersT& parameters, FunctionKind kind,
    FunctionSyntaxKind function_syntax_kind, FunctionBodyType body_type) {
  // 创建一个函数Body解析的Scope
  FunctionBodyParsingScope body_parsing_scope(impl());

  // async, generator 和 module 类型的处理
  if (IsResumableFunction(kind)) impl()->PrepareGeneratorVariables();

  // 创建function_scope 和 inner_scope, 暂时都指向参数的scope
  DeclarationScope* function_scope = parameters.scope;
  DeclarationScope* inner_scope = function_scope;

  if (V8_UNLIKELY(!parameters.is_simple)) { // ...让人猝不及防的否定的否定
    // 当参数is_simple为true(即不带...),需要给函数体初始化一块区域用来存放调用函数时的参数
    if (has_error()) return;
    // 创建一个block,并用传进来的参数初始化
    BlockT init_block = impl()->BuildParameterInitializationBlock(parameters);
    if (IsAsyncFunction(kind) && !IsAsyncGeneratorFunction(kind)) {
      init_block = impl()->BuildRejectPromiseOnException(init_block);
    }
    // 
    body->Add(init_block);
    if (has_error()) return;

    inner_scope = NewVarblockScope();
    inner_scope->set_start_position(scanner()->location().beg_pos);
  }

  // 新建一个内部的body,内容为函数body内的内容
  StatementListT inner_body(pointer_buffer());

  {
    BlockState block_state(&scope_, inner_scope);

    if (body_type == FunctionBodyType::kExpression) {
      // 简单的箭头返回一个值的那种函数的处理
    } else {
      // 定义函数body解析的结束TOKEN
      Token::Value closing_token =
          function_syntax_kind == FunctionSyntaxKind::kWrapped ? Token::EOS
                                                               : Token::RBRACE;

      if (IsAsyncGeneratorFunction(kind)) {
        // ...
      } else if (IsGeneratorFunction(kind)) {
        // ...
      } else if (IsAsyncFunction(kind)) {
        P// ...
      } else {
        // 逻辑进入了ParseStatementList了,就和当初进入function的解析前的流程一样,只是这一次的解析结果放入了生成的inner_body里,
        // 这个inner_body后面会再用到
        ParseStatementList(&inner_body, closing_token);
      }
      // ...
      Expect(closing_token);
    }
  }

  scope()->set_end_position(end_position());

  bool allow_duplicate_parameters = false;

  CheckConflictingVarDeclarations(inner_scope);

  if (V8_LIKELY(parameters.is_simple)) {
    // ...
  } else {
    //当函数参数是带“...”的形式时的处理,需要为“...”指定一块区域供函数body里可以访问到
    // ...

    inner_scope->set_end_position(end_position());
    if (inner_scope->FinalizeBlockScope() != nullptr) {
      BlockT inner_block = factory()->NewBlock(true, inner_body);
      inner_body.Rewind();
      inner_body.Add(inner_block);
      inner_block->set_scope(inner_scope);
      impl()->RecordBlockSourceRange(inner_block, scope()->end_position());
      if (!impl()->HasCheckedSyntax()) {
        const AstRawString* conflict = inner_scope->FindVariableDeclaredIn(
            function_scope, VariableMode::kLastLexicalVariableMode);
        if (conflict != nullptr) {
          impl()->ReportVarRedeclarationIn(conflict, inner_scope);
        }
      }
      impl()->InsertShadowingVarBindingInitializers(inner_block);
    }
  }

  ValidateFormalParameters(language_mode(), parameters,
                           allow_duplicate_parameters);

  if (!IsArrowFunction(kind)) {
    // 非箭头函数,需要在函数的function_scope里对函数体的参数声明作处理,比如有些参数需要hoist之类的
    function_scope->DeclareArguments(ast_value_factory());
  }

  // 猜想这里是函数内部可能会递归调用自己,所以需要在这里function_scope里再声明一次这个函数?
  impl()->DeclareFunctionNameVar(function_name, function_syntax_kind,
                                 function_scope);
  // 将inner_body里的信息整合入(父)body,至此完成对函数体的解析
  inner_body.MergeInto(body);
}

```