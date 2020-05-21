## 作用
V8 用Scope类来实现 js 的“环境”;
比如一个函数A内的变量都声明在该函数A的一个Scope里,如果其中一个变量Aa也是函数,那么该函数Aa自身又会创建一个Scope用来保存Aa内部的变量声明,A是Aa的outer_scope,Aa是A的inner_scope

## Scope类  与 辅助类
```cpp
// src/ast/scopes.h
// 被Scope类用来存储实际声明变量信息的HashMap结构,提供声明(Declare),查找(Lookup),移除(Remove)变量操作;
// Add应该是声明的一种变体,即变量的结构信息(Variable)已经初始化了,可以直接加进当前HashMap
class VariableMap : public ZoneHashMap {
 public:
  explicit VariableMap(Zone* zone);

  Variable* Declare(Zone* zone, Scope* scope, const AstRawString* name,
                    VariableMode mode, VariableKind kind,
                    InitializationFlag initialization_flag,
                    MaybeAssignedFlag maybe_assigned_flag,
                    IsStaticFlag is_static_flag, bool* was_added);

  V8_EXPORT_PRIVATE Variable* Lookup(const AstRawString* name);
  void Remove(Variable* var);
  void Add(Zone* zone, Variable* var);
};

class V8_EXPORT_PRIVATE Scope : public NON_EXPORTED_BASE(ZoneObject) {
public:
  // Construction
  Scope(Zone* zone, Scope* outer_scope, ScopeType scope_type);

  // 在本地scope中查找一个变量名;
  // 直接调用VariableMap的Lookup方法
  Variable* LookupLocal(const AstRawString* name) {
    DCHECK(scope_info_.is_null());
    return variables_.Lookup(name);
  }

  // 在当前scope声明一个变量
  Variable* DeclareLocal(const AstRawString* name, VariableMode mode,
                         VariableKind kind, bool* was_added,
                         InitializationFlag init_flag = kCreatedInitialized);
  // 声明(并定义)一个变量,这个方法包含寻找本地Scope和Scope链是否存在该变量名,并在链上合适的Scope声明该变量
  Variable* DeclareVariable(Declaration* declaration, const AstRawString* name,
                            int pos, VariableMode mode, VariableKind kind,
                            InitializationFlag init, bool* was_added,
                            bool* sloppy_mode_block_scope_function_redefinition,
                            bool* ok);
private:
  // 实际存放当前Scope内变量声明的HashMap结构
  VariableMap variables_;

}
```

### Construction
```cpp
// src/ast/scopes.cc
// 参数zone是一块内存区域的封装结构,用来实际存放变量声明信息
// scope_type用来区分各种Scope类型,如类Scope,函数Scope等
Scope::Scope(Zone* zone, Scope* outer_scope, ScopeType scope_type)
    : zone_(zone),
      outer_scope_(outer_scope), // 初始化outer_scope_属性指向传进来的outer_scope
      variables_(zone),
      scope_type_(scope_type) {
  // 初始化当前Scope的一些属性
  SetDefaults();
  // 严格/非严格模式设置
  set_language_mode(outer_scope->language_mode());
  // 当outer_scope是一个“类”时,需要设置个标记,以便在Scope链里查找变量时应跳过outer_scope(即ClassScope)
  private_name_lookup_skips_outer_class_ =
      outer_scope->is_class_scope() &&
      outer_scope->AsClassScope()->IsParsingHeritage();
  // 初始化innerScope为自身
  outer_scope_->AddInnerScope(this);
}
```

#### ScopeType
```cpp
// src/common/globals.h
enum ScopeType : uint8_t {
  CLASS_SCOPE,     // The scope introduced by a class.
  EVAL_SCOPE,      // The top-level scope for an eval source.
  FUNCTION_SCOPE,  // The top-level scope for a function.
  MODULE_SCOPE,    // The scope introduced by a module literal
  SCRIPT_SCOPE,    // The top-level scope for a script or a top-level eval.
  CATCH_SCOPE,     // The scope introduced by catch.
  BLOCK_SCOPE,     // The scope introduced by a new block.
  WITH_SCOPE       // The scope introduced by with.
};
```

### DeclareLocal
```cpp
// src/ast/scopes.cc
// DeclareLocal 的参数里只有变量的名字和一些其他的辅助信息,并没有变量的内容;
// 因此Declare就是在本地VariableMap里先初始化一块区域(结构)占个位置,并返回指向这个位置的指针,后面就可以用这个指针来把具体定义的内容加进去
Variable* Scope::DeclareLocal(const AstRawString* name, VariableMode mode,
                              VariableKind kind, bool* was_added,
                              InitializationFlag init_flag) {
  // 调用Declare方法将变量的声明加入到VariableMap中
  Variable* var =
      Declare(zone(), name, mode, kind, init_flag, kNotAssigned, was_added);

  // 一些优化处理
  if (is_script_scope() || is_module_scope()) {
    if (mode != VariableMode::kConst) var->SetMaybeAssigned();
    var->set_is_used();
  }
  // 返回变量声明后存放变量内容的地址指针
  return var;
}
```
#### Declare方法
```cpp
  // src/ast/scopes.h
  Variable* Declare(Zone* zone, const AstRawString* name, VariableMode mode,
                    VariableKind kind, InitializationFlag initialization_flag,
                    MaybeAssignedFlag maybe_assigned_flag, bool* was_added) {
    // Static variables can only be declared using ClassScope methods.
    Variable* result = variables_.Declare(
        zone, this, name, mode, kind, initialization_flag, maybe_assigned_flag,
        IsStaticFlag::kNotStatic, was_added);
    // TODO 这个was_added 是什么意义?
    if (*was_added) locals_.Add(result);
    return result;
  }
```

### DeclareVariable
```cpp
// src/ast/scopes.cc
Variable* Scope::DeclareVariable(
    Declaration* declaration, const AstRawString* name, int pos,
    VariableMode mode, VariableKind kind, InitializationFlag init,
    bool* was_added, bool* sloppy_mode_block_scope_function_redefinition,
    bool* ok) {
  
  // 用var 的形式声明一个变量时,如果当前Scope不是合适的可以声明var类型的scope时(即!is_declaration_scope),需要向外一直找到可以用var声明变量的scope,再在那个scope作变量声明操作
  if (mode == VariableMode::kVar && !is_declaration_scope()) {
    return GetDeclarationScope()->DeclareVariable(
        declaration, name, pos, mode, kind, init, was_added,
        sloppy_mode_block_scope_function_redefinition, ok);
  }

  // 在当前scope寻找变量名的声明,如果没声明过,则返回nullptr
  Variable* var = LookupLocal(name);
  // Declare the variable in the declaration scope.
  *was_added = var == nullptr;
  if (V8_LIKELY(*was_added)) {
    // 变量没有在当前Scope声明过
    if (V8_UNLIKELY(is_eval_scope() && is_sloppy(language_mode()) &&
                    mode == VariableMode::kVar)) {
      // ... 一些历史不太合理的语法的兼容
    } else {
      // 调用DeclaraLocal声明变量
      var = DeclareLocal(name, mode, kind, was_added, init);
      DCHECK(*was_added);
    }
  } else {
    // 该变量已经在当前Scope声明过,则不需要再重新声明了
    var->SetMaybeAssigned();
    // ...
  }
  DCHECK_NOT_NULL(var);

  // TODO 这个declaration是指的什么?
  decls_.Add(declaration);
  declaration->set_var(var);
  return var;
}

```



