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
      outer_scope_(outer_scope),
      variables_(zone),
      scope_type_(scope_type) {
  // 初始化当前Scope的一些属性
  SetDefaults();
  // 严格/非严格模式设置
  set_language_mode(outer_scope->language_mode());
  // 当outer_scope是一个“类”时,需要设置个标记,在当前Scope里查找变量时应跳过outer_scope(即ClassScope)
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

### 



