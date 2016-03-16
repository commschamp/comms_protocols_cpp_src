# Appendix E - AlignedUnion

Implementation of `AlignedUnion` type.
```cpp
template <typename TType, typename... TTypes>
class AlignedUnion
{
    typedef typename AlignedUnion<TTypes...>::Type OtherStorage;
    static const std::size_t OtherSize = sizeof(OtherStorage);
    static const std::size_t OtherAlignment = std::alignment_of<OtherStorage>::value;
    typedef typename AlignedUnion<TType>::Type FirstStorage;
    static const std::size_t FirstSize = sizeof(FirstStorage);
    static const std::size_t FirstAlignment = std::alignment_of<FirstStorage>::value;
    static const std::size_t MaxSize = FirstSize > OtherSize ? FirstSize : OtherSize;
    static const std::size_t MaxAlignment = FirstAlignment > OtherAlignment ? FirstAlignment : OtherAlignment;
public:

    /// Type that has proper size and proper alignment to keep any of the
    /// specified types
    typedef typename std::aligned_storage<MaxSize, MaxAlignment>::type Type;
};

template <typename TType>
class AlignedUnion<TType>
{
public:
    typedef typename std::aligned_storage<sizeof(TType), std::alignment_of<TType>::value>::type Type;
};
```
