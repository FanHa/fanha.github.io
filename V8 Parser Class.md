## Class
Class 在ES标准中是个比较新的实现(ES6?)

## 入口
```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseStatementListItem() {
  switch (peek()) {
    // ...
    case Token::CLASS:
      Consume(Token::CLASS);
      return ParseClassDeclaration(nullptr, false);
    // ...
  }
  // ...
}
```

```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::StatementT ParserBase<Impl>::ParseClassDeclaration(
    ZonePtrList<const AstRawString>* names, bool default_export) {
  int class_token_pos = position();
  // Class名暂时初始化为Null
  IdentifierT name = impl()->NullIdentifier();
  // 检测Class后面接的是不是系统保留TOKEN
  bool is_strict_reserved = Token::IsStrictReservedWord(peek());
  // 变量名暂时也设为null
  IdentifierT variable_name = impl()->NullIdentifier();
  if (default_export && (peek() == Token::EXTENDS || peek() == Token::LBRACE)) {
    // Class作为export的结构时需要生成一个默认的名字
    impl()->GetDefaultStrings(&name, &variable_name);
  } else {
    // 非export时, 解析Class后面的token,得到类名
    name = ParseIdentifier();
    variable_name = name;
  }

  ExpressionParsingScope no_expression_scope(impl());
  // 解析Class的内容
  ExpressionT value = ParseClassLiteral(name, scanner()->location(),
                                        is_strict_reserved, class_token_pos);
  no_expression_scope.ValidateExpression();
  int end_pos = position();
  // 在当前scope声明Class
  return impl()->DeclareClass(variable_name, value, names, class_token_pos,
                              end_pos);
}
```
```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::ExpressionT ParserBase<Impl>::ParseClassLiteral(
    IdentifierT name, Scanner::Location class_name_location,
    bool name_is_strict_reserved, int class_token_pos) {
  // 当类的名字为null时,设置标记 匿名
  bool is_anonymous = impl()->IsNull(name);

  // 一些检测,Class 作为一个较新的语法,不应该兼容以前一些“不严格,不优雅”的语法
  if (!impl()->HasCheckedSyntax() && !is_anonymous) {
    if (name_is_strict_reserved) {
      impl()->ReportMessageAt(class_name_location,
                              MessageTemplate::kUnexpectedStrictReserved);
      return impl()->FailureExpression();
    }
    if (impl()->IsEvalOrArguments(name)) {
      impl()->ReportMessageAt(class_name_location,
                              MessageTemplate::kStrictEvalArguments);
      return impl()->FailureExpression();
    }
  }

  // 新建一个ClassScope
  ClassScope* class_scope = NewClassScope(scope(), is_anonymous);
  BlockState block_state(&scope_, class_scope);
  // 同上,Class作为一个较新的语法,不应该兼容以前一些“不严格,不优雅”的语法,直接把languageMode升为‘严格’
  RaiseLanguageMode(LanguageMode::kStrict);

  // 以当前的环境信息为蓝本,初始化一个ClassInfo结构,保存解析初的Class信息,供后面优化整理
  ClassInfo class_info(this);
  class_info.is_anonymous = is_anonymous;

  scope()->set_start_position(end_position());
  if (Check(Token::EXTENDS)) {
    // 如果类是继承自别的类,需要适当被继承类的信息
    ClassScope::HeritageParsingScope heritage(class_scope);
    FuncNameInferrerState fni_state(&fni_);
    ExpressionParsingScope scope(impl());
    // 将继承的类的信息设置为classInfo的entends属性;
    // 这里的‘LeftHandSide’已经暗暗表明了entend只需要知道一个地址,即知道从哪里去找extend的内容,不需要知道extend到底是什么内容
    class_info.extends = ParseLeftHandSideExpression();
    scope.ValidateExpression();
  }

  // Class的Body以‘(’开始
  Expect(Token::LBRACE);

  // 定义一个标记信息便于后面解析时知道当前类是否是某个类的继承
  const bool has_extends = !impl()->IsNull(class_info.extends);

  while (peek() != Token::RBRACE) { // 循环解题Class Body 直至Body的结束标记
    // 跳过js的‘;’语法...
    if (Check(Token::SEMICOLON)) continue;
    FuncNameInferrerState fni_state(&fni_);
    // 如果还没结果过constructor,假设当前要解析的语句是个constructor函数
    bool is_constructor = !class_info.has_seen_constructor;
    
    ParsePropertyInfo prop_info(this);
    prop_info.position = PropertyPosition::kClassLiteral;
    ClassLiteralPropertyT property =
        ParseClassPropertyDefinition(&class_info, &prop_info, has_extends);

    if (has_error()) return impl()->FailureExpression();

    ClassLiteralProperty::Kind property_kind =
        ClassPropertyKindFor(prop_info.kind);
    if (!class_info.has_static_computed_names && prop_info.is_static &&
        prop_info.is_computed_name) {
      class_info.has_static_computed_names = true;
    }
    is_constructor &= class_info.has_seen_constructor;

    bool is_field = property_kind == ClassLiteralProperty::FIELD;

    if (V8_UNLIKELY(prop_info.is_private)) {
      DCHECK(!is_constructor);
      class_info.requires_brand |= (!is_field && !prop_info.is_static);
      class_info.has_private_methods |=
          property_kind == ClassLiteralProperty::METHOD;
      impl()->DeclarePrivateClassMember(class_scope, prop_info.name, property,
                                        property_kind, prop_info.is_static,
                                        &class_info);
      impl()->InferFunctionName();
      continue;
    }

    if (V8_UNLIKELY(is_field)) {
      DCHECK(!prop_info.is_private);
      if (prop_info.is_computed_name) {
        class_info.computed_field_count++;
      }
      impl()->DeclarePublicClassField(class_scope, property,
                                      prop_info.is_static,
                                      prop_info.is_computed_name, &class_info);
      impl()->InferFunctionName();
      continue;
    }

    impl()->DeclarePublicClassMethod(name, property, is_constructor,
                                     &class_info);
    impl()->InferFunctionName();
  }

  Expect(Token::RBRACE);
  int end_pos = end_position();
  class_scope->set_end_position(end_pos);

  VariableProxy* unresolvable = class_scope->ResolvePrivateNamesPartially();
  if (unresolvable != nullptr) {
    impl()->ReportMessageAt(Scanner::Location(unresolvable->position(),
                                              unresolvable->position() + 1),
                            MessageTemplate::kInvalidPrivateFieldResolution,
                            unresolvable->raw_name());
    return impl()->FailureExpression();
  }

  if (class_info.requires_brand) {
    // TODO(joyee): implement static brand checking
    class_scope->DeclareBrandVariable(
        ast_value_factory(), IsStaticFlag::kNotStatic, kNoSourcePosition);
  }

  bool should_save_class_variable_index =
      class_scope->should_save_class_variable_index();
  if (!is_anonymous || should_save_class_variable_index) {
    impl()->DeclareClassVariable(class_scope, name, &class_info,
                                 class_token_pos);
    if (should_save_class_variable_index) {
      class_scope->class_variable()->set_is_used();
      class_scope->class_variable()->ForceContextAllocation();
    }
  }

  return impl()->RewriteClassLiteral(class_scope, name, &class_info,
                                     class_token_pos, end_pos);
}

```
