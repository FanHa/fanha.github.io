## JS变量声明方式(严格模式)
+   let
+   var
+   const

## 
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
  // 执行变量声明的解析,并将信息写入&parsing_result
  ParseVariableDeclarations(var_context, &parsing_result, names);
  // js 的 语句“;”结尾处理
  ExpectSemicolon();
  // 将声明的解析结果绑定到合适的scope(block)里?
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

  // 定义变量需要声明在哪个scope里;
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
    // 判断下一个token是不是合规的Identifier
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
      // 声明变量有时直接后接 ‘=’直接定义变量的内容,也有不解‘=’只占声明位置,暂时不定义
      if (peek() == Token::ASSIGN ||
          (var_context == kForStatement && PeekInOrOf()) ||
          parsing_result->descriptor.mode == VariableMode::kLet) {
        // Assignments need the variable expression for the assignment LHS, and
        // for of/in will need it later, so create the expression now.
        pattern = impl()->ExpressionFromIdentifier(name, decl_pos);
      } else {
        // Otherwise, elide the variable expression and just declare it.
        impl()->DeclareIdentifier(name, decl_pos);
        pattern = impl()->NullExpression();
      }
    } else {
      name = impl()->NullIdentifier();
      pattern = ParseBindingPattern();
      DCHECK(!impl()->IsIdentifier(pattern));
    }

    Scanner::Location variable_loc = scanner()->location();

    ExpressionT value = impl()->NullExpression();
    int value_beg_pos = kNoSourcePosition;
    if (Check(Token::ASSIGN)) {
      DCHECK(!impl()->IsNull(pattern));
      {
        value_beg_pos = peek_position();
        AcceptINScope scope(this, var_context != kForStatement);
        value = ParseAssignmentExpression();
      }
      variable_loc.end_pos = end_position();

      if (!parsing_result->first_initializer_loc.IsValid()) {
        parsing_result->first_initializer_loc = variable_loc;
      }

      // Don't infer if it is "a = function(){...}();"-like expression.
      if (impl()->IsIdentifier(pattern)) {
        if (!value->IsCall() && !value->IsCallNew()) {
          fni_.Infer();
        } else {
          fni_.RemoveLastFunction();
        }
      }

      impl()->SetFunctionNameFromIdentifierRef(value, pattern);
    } else {
#ifdef DEBUG
      // We can fall through into here on error paths, so don't DCHECK those.
      if (!has_error()) {
        // We should never get identifier patterns for the non-initializer path,
        // as those expressions should be elided.
        DCHECK_EQ(!impl()->IsNull(name),
                  Token::IsAnyIdentifier(scanner()->current_token()));
        DCHECK_IMPLIES(impl()->IsNull(pattern), !impl()->IsNull(name));
        // The only times we have a non-null pattern are:
        //   1. This is a destructuring declaration (with no initializer, which
        //      is immediately an error),
        //   2. This is a declaration in a for in/of loop, or
        //   3. This is a let (which has an implicit undefined initializer)
        DCHECK_IMPLIES(
            !impl()->IsNull(pattern),
            !impl()->IsIdentifier(pattern) ||
                (var_context == kForStatement && PeekInOrOf()) ||
                parsing_result->descriptor.mode == VariableMode::kLet);
      }
#endif

      if (var_context != kForStatement || !PeekInOrOf()) {
        // ES6 'const' and binding patterns require initializers.
        if (parsing_result->descriptor.mode == VariableMode::kConst ||
            impl()->IsNull(name)) {
          impl()->ReportMessageAt(
              Scanner::Location(decl_pos, end_position()),
              MessageTemplate::kDeclarationMissingInitializer,
              impl()->IsNull(name) ? "destructuring" : "const");
          return;
        }
        // 'let x' initializes 'x' to undefined.
        if (parsing_result->descriptor.mode == VariableMode::kLet) {
          value = factory()->NewUndefinedLiteral(position());
        }
      }
    }

    int initializer_position = end_position();
    auto declaration_end = target_scope->declarations()->end();
    for (; declaration_it != declaration_end; ++declaration_it) {
      declaration_it->var()->set_initializer_position(initializer_position);
    }

    // Patterns should be elided iff. they don't have an initializer.
    DCHECK_IMPLIES(impl()->IsNull(pattern),
                   impl()->IsNull(value) ||
                       (var_context == kForStatement && PeekInOrOf()));

    typename DeclarationParsingResult::Declaration decl(pattern, value);
    decl.value_beg_pos = value_beg_pos;

    parsing_result->declarations.push_back(decl);
  } while (Check(Token::COMMA));

  parsing_result->bindings_loc =
      Scanner::Location(bindings_start, end_position());
}
```
