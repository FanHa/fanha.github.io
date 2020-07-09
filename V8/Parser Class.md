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
### 解析Class内容
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
  // 同上,Class作为一个较新的语法,不想兼容以前一些“不严格,不优雅”的语法,直接把languageMode升为‘严格’
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
    // 初始化化类的属性信息
    ParsePropertyInfo prop_info(this);
    prop_info.position = PropertyPosition::kClassLiteral;

    // 解析类里面的单个property;
    // 一个while循环解析一条property;
    ClassLiteralPropertyT property =
        ParseClassPropertyDefinition(&class_info, &prop_info, has_extends);

    if (has_error()) return impl()->FailureExpression();

    // 从前面的解析结果中取出类的属性;
    // 类的属性有 getter,setter,method,field 四种
    ClassLiteralProperty::Kind property_kind =
        ClassPropertyKindFor(prop_info.kind);
    // 根据当前property解析结果决定设置classInfo的has_static_computed_names属性
    if (!class_info.has_static_computed_names && prop_info.is_static &&
        prop_info.is_computed_name) {
      class_info.has_static_computed_names = true;
    }
    // 设置当前property是不是constructor
    is_constructor &= class_info.has_seen_constructor;
    // 设置当前property是不是field
    bool is_field = property_kind == ClassLiteralProperty::FIELD;

    if (V8_UNLIKELY(prop_info.is_private)) {
      // 新的ES标准将会支持类的私有属性,私有属性的声明和传统的属性声明走不同的流程处理,声明完后continue到下一个循环
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
      // 当前property是field时,声明该属性,并continue到下一个循环
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

    // 声明公共方法
    impl()->DeclarePublicClassMethod(name, property, is_constructor,
                                     &class_info);
    impl()->InferFunctionName();
  }
  // Class Body解析完
  Expect(Token::RBRACE);
  int end_pos = end_position();
  class_scope->set_end_position(end_pos);
  // ...
  // 需要重新整理下Class的解析结果,生成一个新的结构更清晰的AST节点并返回
  return impl()->RewriteClassLiteral(class_scope, name, &class_info,
                                     class_token_pos, end_pos);
}

```

#### ParseClassPropertyDefinition
解析Class Body里的单条Proerty
```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::ClassLiteralPropertyT
ParserBase<Impl>::ParseClassPropertyDefinition(ClassInfo* class_info,
                                               ParsePropertyInfo* prop_info,
                                               bool has_extends) {
  // 取出Property的名字
  Token::Value name_token = peek();
  int property_beg_pos = scanner()->peek_location().beg_pos;
  int name_token_position = property_beg_pos;
  ExpressionT name_expression;
  if (name_token == Token::STATIC) {
    // Property带 ‘static’标记的情况
    Consume(Token::STATIC);
    name_token_position = scanner()->peek_location().beg_pos;
    if (peek() == Token::LPAREN) {
      // 一些不常见的用法?
      //...
    } else if (peek() == Token::ASSIGN || peek() == Token::SEMICOLON ||
               peek() == Token::RBRACE) {
      //...
    } else {
      // 设置propInfo的is_static标记,然后调用ParseProperty解析当前property信息
      prop_info->is_static = true;
      // 解析Property名字;
      // async, *, public, private , 数字转换等等等等
      name_expression = ParseProperty(prop_info);
    }
  } else {
    name_expression = ParseProperty(prop_info);
  }

  // 设置ClassInfo的has_name_static_property属性;
  // 猜想如果一个类没有任何statc类型的Property,v8可以作些优化
  if (!class_info->has_name_static_property && prop_info->is_static &&
      impl()->IsName(prop_info->name)) {
    class_info->has_name_static_property = true;
  }

  switch (prop_info->kind) {
    case ParsePropertyKind::kAssign:
    case ParsePropertyKind::kClassField:
    case ParsePropertyKind::kShorthandOrClassField:
    case ParsePropertyKind::kNotSet: {
      // 普通Field类型的Property
      // 几种常见的,兼容的,未来的类型等等统一设置成ClassField类型
      prop_info->kind = ParsePropertyKind::kClassField;
      DCHECK_IMPLIES(prop_info->is_computed_name, !prop_info->is_private);

      if (!prop_info->is_computed_name) {
        CheckClassFieldName(prop_info->name, prop_info->is_static);
      }
      // 解析Class的成员(Member)的内容
      ExpressionT initializer = ParseMemberInitializer(
          class_info, property_beg_pos, prop_info->is_static);
      // 忽略‘:’token
      ExpectSemicolon();
      // 生成Property的解析结果
      ClassLiteralPropertyT result = factory()->NewClassLiteralProperty(
          name_expression, initializer, ClassLiteralProperty::FIELD,
          prop_info->is_static, prop_info->is_computed_name,
          prop_info->is_private);
      // 给解析结果冠个名
      impl()->SetFunctionNameFromPropertyName(result, prop_info->name);

      return result;
    }
    case ParsePropertyKind::kMethod: {// Class方法类型的解析
      if (!prop_info->is_computed_name) {
        CheckClassMethodName(prop_info->name, ParsePropertyKind::kMethod,
                             prop_info->function_flags, prop_info->is_static,
                             &class_info->has_seen_constructor);
      }
      // 取出前面解析的一些更细节的方法信息,比如async, *;
      FunctionKind kind = MethodKindFor(prop_info->function_flags);

      if (!prop_info->is_static && impl()->IsConstructor(prop_info->name)) {
        // 判断当前方法名是Construcotr函数,同时当前方法不是static时,
        // 则认为当前的方法是Constructor,需要设置ClassInfo的has_ssen_constructor为true
        class_info->has_seen_constructor = true;
        // 同时根据类是否是extend需要设置当前Constructor的kind 是Base 还是 Derived
        kind = has_extends ? FunctionKind::kDerivedConstructor
                           : FunctionKind::kBaseConstructor;
      }

      // 解析方法里的内容
      ExpressionT value = impl()->ParseFunctionLiteral(
          prop_info->name, scanner()->location(), kSkipFunctionNameCheck, kind,
          name_token_position, FunctionSyntaxKind::kAccessorOrMethod,
          language_mode(), nullptr);

      // 生成Property的解析结果
      ClassLiteralPropertyT result = factory()->NewClassLiteralProperty(
          name_expression, value, ClassLiteralProperty::METHOD,
          prop_info->is_static, prop_info->is_computed_name,
          prop_info->is_private);
      // 给解析结果冠个名
      impl()->SetFunctionNameFromPropertyName(result, prop_info->name);
      return result;
    }

    case ParsePropertyKind::kAccessorGetter:
    case ParsePropertyKind::kAccessorSetter: {
      // Class 的 ’getter' 和 ‘setter’方法的解析
      bool is_get = prop_info->kind == ParsePropertyKind::kAccessorGetter;
      //...
      // 判断是 ‘getter’ 方法还是 ‘setter’ 方法
      FunctionKind kind = is_get ? FunctionKind::kGetterFunction
                                 : FunctionKind::kSetterFunction;

      // 解析方法内容
      FunctionLiteralT value = impl()->ParseFunctionLiteral(
          prop_info->name, scanner()->location(), kSkipFunctionNameCheck, kind,
          name_token_position, FunctionSyntaxKind::kAccessorOrMethod,
          language_mode(), nullptr);

      ClassLiteralProperty::Kind property_kind =
          is_get ? ClassLiteralProperty::GETTER : ClassLiteralProperty::SETTER;
      // 生成Property的解析结果
      ClassLiteralPropertyT result = factory()->NewClassLiteralProperty(
          name_expression, value, property_kind, prop_info->is_static,
          prop_info->is_computed_name, prop_info->is_private);
      const AstRawString* prefix =
          is_get ? ast_value_factory()->get_space_string()
                 : ast_value_factory()->set_space_string();
      // 给解析结果冠个名
      impl()->SetFunctionNameFromPropertyName(result, prop_info->name, prefix);
      return result;
    }
    case ParsePropertyKind::kValue:
    case ParsePropertyKind::kShorthand:
    case ParsePropertyKind::kSpread:
      // 其他现在认为是语法错误的处理
      impl()->ReportUnexpectedTokenAt(
          Scanner::Location(name_token_position, name_expression->position()),
          name_token);
      return impl()->NullLiteralProperty();
  }
  UNREACHABLE();
}
```

##### ParseMemberInitializer

#### DeclarePrivateClassMember
```cpp
// src/parsing/parser.cc
void Parser::DeclarePrivateClassMember(ClassScope* scope,
                                       const AstRawString* property_name,
                                       ClassLiteralProperty* property,
                                       ClassLiteralProperty::Kind kind,
                                       bool is_static, ClassInfo* class_info) {
  if (kind == ClassLiteralProperty::Kind::FIELD) {
    // 根据Property是否为static类型将Property加到ClassInfo的不同fields里;
    // static_fields用于类;
    // instance_fields用于由类生成的实例
    if (is_static) {
      class_info->static_fields->Add(property, zone());
    } else {
      class_info->instance_fields->Add(property, zone());
    }
  }

  // 创建一个用来保存私有非匿名变量的Varaible结构,即在scope(ClassScope)里作变量声明;
  // 并将信息设置如Property
  Variable* private_name_var = CreatePrivateNameVariable(
      scope, GetVariableMode(kind),
      is_static ? IsStaticFlag::kStatic : IsStaticFlag::kNotStatic,
      property_name);
  int pos = property->value()->position();
  if (pos == kNoSourcePosition) {
    pos = property->key()->position();
  }
  private_name_var->set_initializer_position(pos);
  property->set_private_name_var(private_name_var);
  // 将Property加入到ClassInfo的private_members里
  class_info->private_members->Add(property, zone());
}

```

#### DeclarePublicClassField
```cpp
// src/parsing/parser.cc
void Parser::DeclarePublicClassField(ClassScope* scope,
                                     ClassLiteralProperty* property,
                                     bool is_static, bool is_computed_name,
                                     ClassInfo* class_info) {
  // 由is_static标记决定加入不同的容器
  if (is_static) {
    class_info->static_fields->Add(property, zone());
  } else {
    class_info->instance_fields->Add(property, zone());
  }
  // ...
}
```

#### DeclarePublicClassMethod
constructor和其他public method都走这一流程
```cpp
// src/parsing/parser.cc
void Parser::DeclarePublicClassMethod(const AstRawString* class_name,
                                      ClassLiteralProperty* property,
                                      bool is_constructor,
                                      ClassInfo* class_info) {
  if (is_constructor) {
    // constructor的情况,在ClassInfo里设置constructor为当前Property
    class_info->constructor = property->value()->AsFunctionLiteral();
    // 规范统一下constructor的名字
    class_info->constructor->set_raw_name(
        class_name != nullptr ? ast_value_factory()->NewConsString(class_name)
                              : nullptr);
    return;
  }
  // 普通的public method 直接把Property加入到ClassInfo的public_members容器里
  class_info->public_members->Add(property, zone());
}
```

#### RewriteClassLiteral
整理前面解析出来的ClassInfo, 优化生成一个新的Class Ast节点
```cpp
// src/parsing/parser.cc
Expression* Parser::RewriteClassLiteral(ClassScope* block_scope,
                                        const AstRawString* name,
                                        ClassInfo* class_info, int pos,
                                        int end_pos) 

  bool has_extends = class_info->extends != nullptr;
  bool has_default_constructor = class_info->constructor == nullptr;
  // 没有显式知名constructor时生成一个默认的constructor方法
  if (has_default_constructor) {
    class_info->constructor =
        DefaultConstructor(name, has_extends, pos, end_pos);
  }

  if (name != nullptr) {
    DCHECK_NOT_NULL(block_scope->class_variable());
    block_scope->class_variable()->set_initializer_position(end_pos);
  }

  FunctionLiteral* static_fields_initializer = nullptr;
  if (class_info->has_static_class_fields) {
    static_fields_initializer = CreateInitializerFunction(
        "<static_fields_initializer>", class_info->static_fields_scope,
        class_info->static_fields);
  }

  FunctionLiteral* instance_members_initializer_function = nullptr;
  if (class_info->has_instance_members) {
    instance_members_initializer_function = CreateInitializerFunction(
        "<instance_members_initializer>", class_info->instance_members_scope,
        class_info->instance_fields);
    class_info->constructor->set_requires_instance_members_initializer(true);
    class_info->constructor->add_expected_properties(
        class_info->instance_fields->length());
  }
  // 根据前面的信息生成一个AST节点并返回
  ClassLiteral* class_literal = factory()->NewClassLiteral(
      block_scope, class_info->extends, class_info->constructor,
      class_info->public_members, class_info->private_members,
      static_fields_initializer, instance_members_initializer_function, pos,
      end_pos, class_info->has_name_static_property,
      class_info->has_static_computed_names, class_info->is_anonymous,
      class_info->has_private_methods);

  AddFunctionForNameInference(class_info->constructor);
  return class_literal;
}

```