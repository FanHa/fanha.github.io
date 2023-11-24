# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
每一个filter都需要调用`decodeHeaders`方法,decodeHeaders 的第一个参数 headers是一个内存地址,存储的事请求的头信息,供filter读取和修改,headers的数据结构是`Http::RequestHeaderMap`
```cpp
Http::FilterHeadersStatus Filter::decodeHeaders(Http::RequestHeaderMap& headers, bool end_stream) {
  downstream_headers_ = &headers;
  // ...
}
```
```cpp
// source/common/http/header_map_impl.h
/**
 * Implementation of Http::HeaderMap. This is heavily optimized for performance. Roughly, when
 * headers are added to the map by string, we do a trie lookup to see if it's one of the O(1)
 * headers. If it is, we store a reference to it that can be accessed later directly via direct
 * method access. Most high performance paths use O(1) direct method access. In general, we try to
 * copy as little as possible and allocate as little as possible in any of the paths. 
 */
class HeaderMapImpl : NonCopyable {
public:
  virtual ~HeaderMapImpl() = default;

  // The following "constructors" call virtual functions during construction and must use the
  // static factory pattern.
  static void copyFrom(HeaderMap& lhs, const HeaderMap& rhs);
  // The value_type of iterator must be pair, and the first value of them must be LowerCaseString.
  // If not, it won't be compiled successfully.
  template <class It> static void initFromInitList(HeaderMap& new_header_map, It begin, It end) {
    for (auto it = begin; it != end; ++it) {
      static_assert(std::is_same<decltype(it->first), LowerCaseString>::value,
                    "iterator must be pair and the first value of them must be LowerCaseString");
      HeaderString key_string;
      key_string.setCopy(it->first.get().c_str(), it->first.get().size());
      HeaderString value_string;
      value_string.setCopy(it->second.c_str(), it->second.size());
      new_header_map.addViaMove(std::move(key_string), std::move(value_string));
    }
  }

  // Performs a manual byte size count for test verification.
  void verifyByteSizeInternalForTest() const;

  // Note: This class does not actually implement Http::HeaderMap to avoid virtual inheritance in
  // the derived classes. Instead, it is used as a mix-in class for TypedHeaderMapImpl below. This
  // both avoid virtual inheritance and allows the concrete final header maps to use a variable
  // length member at the end.
  bool operator==(const HeaderMap& rhs) const;
  bool operator!=(const HeaderMap& rhs) const;
  void addViaMove(HeaderString&& key, HeaderString&& value);
  void addReference(const LowerCaseString& key, absl::string_view value);
  void addReferenceKey(const LowerCaseString& key, uint64_t value);
  void addReferenceKey(const LowerCaseString& key, absl::string_view value);
  void addCopy(const LowerCaseString& key, uint64_t value);
  void addCopy(const LowerCaseString& key, absl::string_view value);
  void appendCopy(const LowerCaseString& key, absl::string_view value);
  void setReference(const LowerCaseString& key, absl::string_view value);
  void setReferenceKey(const LowerCaseString& key, absl::string_view value);
  void setCopy(const LowerCaseString& key, absl::string_view value);
  uint64_t byteSize() const;
  HeaderMap::GetResult get(const LowerCaseString& key) const;
  void iterate(HeaderMap::ConstIterateCb cb) const;
  void iterateReverse(HeaderMap::ConstIterateCb cb) const;
  void clear();
  size_t remove(const LowerCaseString& key);
  size_t removeIf(const HeaderMap::HeaderMatchPredicate& predicate);
  size_t removePrefix(const LowerCaseString& key);
  size_t size() const { return headers_.size(); }
  bool empty() const { return headers_.empty(); }
  void dumpState(std::ostream& os, int indent_level = 0) const;
  StatefulHeaderKeyFormatterOptConstRef formatter() const {
    return StatefulHeaderKeyFormatterOptConstRef(makeOptRefFromPtr(formatter_.get()));
  }
  StatefulHeaderKeyFormatterOptRef formatter() { return makeOptRefFromPtr(formatter_.get()); }

protected:
  struct HeaderEntryImpl : public HeaderEntry, NonCopyable {
    HeaderEntryImpl(const LowerCaseString& key);
    HeaderEntryImpl(const LowerCaseString& key, HeaderString&& value);
    HeaderEntryImpl(HeaderString&& key, HeaderString&& value);

    // HeaderEntry
    const HeaderString& key() const override { return key_; }
    void value(absl::string_view value) override;
    void value(uint64_t value) override;
    void value(const HeaderEntry& header) override;
    const HeaderString& value() const override { return value_; }
    HeaderString& value() override { return value_; }

    HeaderString key_;
    HeaderString value_;
    std::list<HeaderEntryImpl>::iterator entry_;
  };
  using HeaderNode = std::list<HeaderEntryImpl>::iterator;

  /**
   * This is the static lookup table that is used to determine whether a header is one of the O(1)
   * headers. This uses a trie for lookup time at most equal to the size of the incoming string.
   */
  struct StaticLookupResponse {
    HeaderEntryImpl** entry_;
    const LowerCaseString* key_;
  };

  /**
   * Base class for a static lookup table that converts a string key into an O(1) header.
   */
  template <class Interface>
  struct StaticLookupTable
      : public TrieLookupTable<std::function<StaticLookupResponse(HeaderMapImpl&)>> {
    StaticLookupTable();

    void finalizeTable() {
      CustomInlineHeaderRegistry::finalize<Interface::header_map_type>();
      auto& headers = CustomInlineHeaderRegistry::headers<Interface::header_map_type>();
      size_ = headers.size();
      for (const auto& header : headers) {
        this->add(header.first.get().c_str(), [&header](HeaderMapImpl& h) -> StaticLookupResponse {
          return {&h.inlineHeaders()[header.second], &header.first};
        });
      }
    }

    static size_t size() {
      // The size of the lookup table is finalized when the singleton lookup table is created. This
      // allows for late binding of custom headers as well as envoy header prefix changes. This
      // does mean that once the first header map is created of this type, no further changes are
      // possible.
      // TODO(mattklein123): If we decide to keep this implementation, it is conceivable that header
      // maps could be created by an API factory that is owned by the listener/HCM, thus making
      // O(1) header delivery over xDS possible.
      return ConstSingleton<StaticLookupTable>::get().size_;
    }

    static absl::optional<StaticLookupResponse> lookup(HeaderMapImpl& header_map,
                                                       absl::string_view key) {
      const auto& entry = ConstSingleton<StaticLookupTable>::get().find(key);
      if (entry != nullptr) {
        return entry(header_map);
      } else {
        return absl::nullopt;
      }
    }

    size_t size_;
  };

  /**
   * List of HeaderEntryImpl that keeps the pseudo headers (key starting with ':') in the front
   * of the list (as required by nghttp2) and otherwise maintains insertion order.
   * When the list size is greater or equal to the envoy.http.headermap.lazy_map_min_size runtime
   * feature value (defaults to 3, if not set), all headers are added to a map, to allow
   * fast access given a header key. Once the map is initialized, it will be used even if the number
   * of headers decreases below the threshold.
   *
   * Note: the internal iterators held in fields make this unsafe to copy and move, since the
   * reference to end() is not preserved across a move (see Notes in
   * https://en.cppreference.com/w/cpp/container/list/list). The NonCopyable will suppress both copy
   * and move constructors/assignment.
   * TODO(htuch): Maybe we want this to movable one day; for now, our header map moves happen on
   * HeaderMapPtr, so the performance impact should not be evident.
   */
  class HeaderList : NonCopyable {
  public:
    using HeaderNodeVector = absl::InlinedVector<HeaderNode, 1>;
    using HeaderLazyMap = absl::flat_hash_map<absl::string_view, HeaderNodeVector>;

    HeaderList()
        : pseudo_headers_end_(headers_.end()),
          lazy_map_min_size_(static_cast<uint32_t>(
              Runtime::getInteger("envoy.http.headermap.lazy_map_min_size", 3))) {}

    template <class Key> bool isPseudoHeader(const Key& key) {
      return !key.getStringView().empty() && key.getStringView()[0] == ':';
    }

    template <class Key, class... Value> HeaderNode insert(Key&& key, Value&&... value) {
      const bool is_pseudo_header = isPseudoHeader(key);
      HeaderNode i = headers_.emplace(is_pseudo_header ? pseudo_headers_end_ : headers_.end(),
                                      std::forward<Key>(key), std::forward<Value>(value)...);
      if (!lazy_map_.empty()) {
        lazy_map_[i->key().getStringView()].push_back(i);
      }
      if (!is_pseudo_header && pseudo_headers_end_ == headers_.end()) {
        pseudo_headers_end_ = i;
      }
      return i;
    }

    HeaderNode erase(HeaderNode i, bool remove_from_map) {
      if (pseudo_headers_end_ == i) {
        pseudo_headers_end_++;
      }
      if (remove_from_map) {
        lazy_map_.erase(i->key().getStringView());
      }
      return headers_.erase(i);
    }

    template <class UnaryPredicate> void removeIf(UnaryPredicate p) {
      if (!lazy_map_.empty()) {
        // Lazy map is used, iterate over its elements and remove those that satisfy the predicate
        // from the map and from the list.
        for (auto map_it = lazy_map_.begin(); map_it != lazy_map_.end();) {
          auto& values_vec = map_it->second;
          ASSERT(!values_vec.empty());
          // The following call to std::remove_if removes the elements that satisfy the
          // UnaryPredicate and shifts the vector elements, but does not resize the vector.
          // The call to erase that follows erases the unneeded cells (from remove_pos to the
          // end) and modifies the vector's size.
          const auto remove_pos =
              std::remove_if(values_vec.begin(), values_vec.end(), [&](HeaderNode it) {
                if (p(*(it->entry_))) {
                  // Remove the element from the list.
                  if (pseudo_headers_end_ == it->entry_) {
                    pseudo_headers_end_++;
                  }
                  headers_.erase(it);
                  return true;
                }
                return false;
              });
          values_vec.erase(remove_pos, values_vec.end());

          // If all elements were removed from the map entry, erase it.
          if (values_vec.empty()) {
            lazy_map_.erase(map_it++);
          } else {
            map_it++;
          }
        }
      } else {
        // The lazy map isn't used, iterate over the list elements and remove elements that satisfy
        // the predicate.
        headers_.remove_if([&](const HeaderEntryImpl& entry) {
          const bool to_remove = p(entry);
          if (to_remove) {
            if (pseudo_headers_end_ == entry.entry_) {
              pseudo_headers_end_++;
            }
          }
          return to_remove;
        });
      }
    }

    /*
     * Creates and populates a map if the number of headers is at least the
     * envoy.http.headermap.lazy_map_min_size runtime feature value.
     *
     * @return if a map was created.
     */
    bool maybeMakeMap();

    /*
     * Removes a given key and its values from the HeaderList.
     *
     * @return the number of bytes that were removed.
     */
    size_t remove(absl::string_view key);

    std::list<HeaderEntryImpl>::iterator begin() { return headers_.begin(); }
    std::list<HeaderEntryImpl>::iterator end() { return headers_.end(); }
    std::list<HeaderEntryImpl>::const_iterator begin() const { return headers_.begin(); }
    std::list<HeaderEntryImpl>::const_iterator end() const { return headers_.end(); }
    std::list<HeaderEntryImpl>::const_reverse_iterator rbegin() const { return headers_.rbegin(); }
    std::list<HeaderEntryImpl>::const_reverse_iterator rend() const { return headers_.rend(); }
    HeaderLazyMap::iterator mapFind(absl::string_view key) { return lazy_map_.find(key); }
    HeaderLazyMap::iterator mapEnd() { return lazy_map_.end(); }
    size_t size() const { return headers_.size(); }
    bool empty() const { return headers_.empty(); }
    void clear() {
      headers_.clear();
      pseudo_headers_end_ = headers_.end();
      lazy_map_.clear();
    }

  private:
    std::list<HeaderEntryImpl> headers_;
    HeaderNode pseudo_headers_end_;
    // The number of headers threshold for lazy map usage.
    const uint32_t lazy_map_min_size_;
    HeaderLazyMap lazy_map_;
  };

  void insertByKey(HeaderString&& key, HeaderString&& value);
  static uint64_t appendToHeader(HeaderString& header, absl::string_view data,
                                 absl::string_view delimiter = ",");
  HeaderEntryImpl& maybeCreateInline(HeaderEntryImpl** entry, const LowerCaseString& key);
  HeaderEntryImpl& maybeCreateInline(HeaderEntryImpl** entry, const LowerCaseString& key,
                                     HeaderString&& value);

  HeaderMap::NonConstGetResult getExisting(absl::string_view key);
  size_t removeExisting(absl::string_view key);

  size_t removeInline(HeaderEntryImpl** entry);
  void updateSize(uint64_t from_size, uint64_t to_size);
  void addSize(uint64_t size);
  void subtractSize(uint64_t size);
  virtual absl::optional<StaticLookupResponse> staticLookup(absl::string_view) PURE;
  virtual void clearInline() PURE;
  virtual HeaderEntryImpl** inlineHeaders() PURE;

  HeaderList headers_;
  // TODO(mattklein123): The formatter does not currently get copied when a header map gets
  // copied. This may be problematic in certain cases like request shadowing. This is omitted
  // on purpose until someone asks for it, at which point a clone() method can be created to
  // avoid using extra space/processing for a shared_ptr.
  StatefulHeaderKeyFormatterPtr formatter_;
  // This holds the internal byte size of the HeaderMap.
  uint64_t cached_byte_size_ = 0;
};

```