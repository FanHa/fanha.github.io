## JS变量声明方式(严格模式)
+   let
+   var
+   const

## 变量statement语句解析
```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseStatementListItem() {
  switch (peek()) {
    //...
    // 试探token,为‘var’,‘const’,‘let’时调用ParseVariableStatement
    case Token::VAR:
    case Token::CONST:
      return ParseVariableStatement(kStatementListItem, nullptr);
    case Token::LET:
      if (IsNextLetKeyword()) {
        return ParseVariableStatement(kStatementListItem, nullptr);
      }
      break;
    // ...
  }
  // ...
}

template <typename Impl>
typename ParserBase<Impl>::StatementT ParserBase<Impl>::ParseVariableStatement(
    VariableDeclarationContext var_context,
    ZonePtrList<const AstRawString>* names) {
  
  // 初始化一个变量声明结果结构 parsing_result
  DeclarationParsingResult parsing_result;
  // 执行变量声明的解析,并将解析结果写入&parsing_result
  ParseVariableDeclarations(var_context, &parsing_result, names);
  // js语句的“;”结尾处理
  ExpectSemicolon();
  // 将声明的解析结果初始化一个Block ??
  return impl()->BuildInitializationBlock(&parsing_result);
}

template <typename Impl>
void ParserBase<Impl>::ParseVariableDeclarations(
    VariableDeclarationContext var_context,
    DeclarationParsingResult* parsing_result,
    ZonePtrList<const AstRawString>* names) {
  parsing_result->descriptor.kind = NORMAL_VARIABLE;
  parsing_result->descriptor.declaration_pos = peek_position();
  parsing_result->descriptor.initialization_pos = peek_position();

  // 根据不同的声明方式(var,const,let),给解析结果parsing_result设置mode
  switch (peek()) {
    case Token::VAR:
      parsing_result->descriptor.mode = VariableMode::kVar;
      Consume(Token::VAR);
      break;
    case Token::CONST:
      Consume(Token::CONST);
      DCHECK_NE(var_context, kStatement);
      parsing_result->descriptor.mode = VariableMode::kConst;
      break;
    case Token::LET:
      Consume(Token::LET);
      DCHECK_NE(var_context, kStatement);
      parsing_result->descriptor.mode = VariableMode::kLet;
      break;
    default:
      UNREACHABLE();  // by current callers
      break;
  }

  // 找到变量需要声明在哪个scope里;
  // LexicalMode的判断,这里就是 let/const(IsLexcialVariableMode) 与 var 声明的不同;
  // let/const声明是 “块级”声明,声明所在的scope即为当前的scope;
  // var则需要忽略“块级”scope,往外层一层层寻找,直到合适的scope
  Scope* target_scope = IsLexicalVariableMode(parsing_result->descriptor.mode)
                            ? scope()
                            : scope()->GetDeclarationScope();

  // 创建一个遍历器(it),指向目标scope的声明列表的 end
  auto declaration_it = target_scope->declarations()->end();

  int bindings_start = peek_position();
  do {
    // Parse binding pattern.
    FuncNameInferrerState fni_state(&fni_);

    int decl_pos = peek_position();

    IdentifierT name;
    ExpressionT pattern;
    // 判断下一个token是不是普通的Identifier
    if (V8_LIKELY(Token::IsAnyIdentifier(peek()))) {
      // 取出要声明的变量名
      name = ParseAndClassifyIdentifier(Next());
      // 严格模式下,变量的的重复声明是会报异常的;
      if (V8_UNLIKELY(is_strict(language_mode()) &&
                      impl()->IsEvalOrArguments(name))) {
        impl()->ReportMessageAt(scanner()->location(),
                                MessageTemplate::kStrictEvalArguments);
        return;
      }
      // 声明变量有时直接后接 ‘=’直接定义变量的内容,也有不接‘=’只占声明位置,暂时不定义
      if (peek() == Token::ASSIGN ||
          (var_context == kForStatement && PeekInOrOf()) ||
          parsing_result->descriptor.mode == VariableMode::kLet) {
        // 解析变量名,解析变量名(Identifier)的位置,如果找不到则在当前scope新增一个变量;
        // 猜想这个pattern可能是对变量形式的一种初始化
        pattern = impl()->ExpressionFromIdentifier(name, decl_pos);
      } else {
        // 在合适的scope声明变量
        impl()->DeclareIdentifier(name, decl_pos);
        // 因为只声明,默认的变量的形式是Null
        pattern = impl()->NullExpression();
      }
    } else {
      // 存在解构声明和定义的情况
      name = impl()->NullIdentifier();
      pattern = ParseBindingPattern();
      DCHECK(!impl()->IsIdentifier(pattern));
    }

    Scanner::Location variable_loc = scanner()->location();

    // 对变量的值(value)的初始化
    ExpressionT value = impl()->NullExpression();
    int value_beg_pos = kNoSourcePosition;

    // 如果接下来的token是“=”,表明要对变量进行赋值
    if (Check(Token::ASSIGN)) {
      DCHECK(!impl()->IsNull(pattern));
      {
        value_beg_pos = peek_position();
        AcceptINScope scope(this, var_context != kForStatement);
        // 解析值的表达式
        value = ParseAssignmentExpression();
      }
      variable_loc.end_pos = end_position();

      if (!parsing_result->first_initializer_loc.IsValid()) {
        parsing_result->first_initializer_loc = variable_loc;
      }
      // 这里是对匿名函数的信息的处理
      if (impl()->IsIdentifier(pattern)) {
        if (!value->IsCall() && !value->IsCallNew()) {
          fni_.Infer();
        } else {
          fni_.RemoveLastFunction();
        }
      }

      // 通过前面生成的变量pattern 给值value一个命名;
      // 猜想用于引擎查找该值,或打印调试信息?
      impl()->SetFunctionNameFromIdentifierRef(value, pattern);
    } else {
      if (var_context != kForStatement || !PeekInOrOf()) {
        // const必须有初始值
        if (parsing_result->descriptor.mode == VariableMode::kConst ||
            impl()->IsNull(name)) {
          impl()->ReportMessageAt(
              Scanner::Location(decl_pos, end_position()),
              MessageTemplate::kDeclarationMissingInitializer,
              impl()->IsNull(name) ? "destructuring" : "const");
          return;
        }
        // 用let只声明变量时需要给变量一个undefined的初始值
        if (parsing_result->descriptor.mode == VariableMode::kLet) {
          value = factory()->NewUndefinedLiteral(position());
        }
      }
    }

    // 给新声明的变量Varaible设置初始position属性
    int initializer_position = end_position();
    auto declaration_end = target_scope->declarations()->end();
    for (; declaration_it != declaration_end; ++declaration_it) {
      declaration_it->var()->set_initializer_position(initializer_position);
    }

    // 创建一个声明的描述,并push到解析结果中
    typename DeclarationParsingResult::Declaration decl(pattern, value);
    decl.value_beg_pos = value_beg_pos;

    parsing_result->declarations.push_back(decl);
  } while (Check(Token::COMMA)); // 同时声明多个变量时的处理

  parsing_result->bindings_loc =
      Scanner::Location(bindings_start, end_position());
  // 这个parsing_result是函数的传址参数,解析结果的信息通过属性设置保存在这个result里,供后面的流程使用这个result实装变量的声明和定义
}
```