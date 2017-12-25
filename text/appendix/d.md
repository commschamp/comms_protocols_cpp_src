# Appendix D - tupleForEachType

Implementation of `tupleForEachType()` function. Namespace `details` contains some
helper classes.
```cpp
namespace details
{
template <std::size_t TRem>
class TupleForEachTypeHelper
{
public:
    template <typename TTuple, typename TFunc>
    static void exec(TFunc&& func)
    {
        using Tuple = typename std::decay<TTuple>::type;
        static const std::size_t TupleSize = std::tuple_size<Tuple>::value;
        static_assert(TRem <= TupleSize, "Incorrect TRem");

        static const std::size_t Idx = TupleSize - TRem;
        using ElemType = typename std::tuple_element<Idx, Tuple>::type;
        func.template operator()<ElemType>();
        TupleForEachTypeHelper<TRem - 1>::template exec<TTuple>(
            std::forward<TFunc>(func));
    }
};

template <>
class TupleForEachTypeHelper<0>
{

public:
    template <typename TTuple, typename TFunc>
    static void exec(TFunc&& func)
    {
        // Nothing to do
    }
};
}  // namespace details

template <typename TTuple, typename TFunc>
void tupleForEachType(TFunc&& func)
{
    using Tuple = typename std::decay<TTuple>::type;
    static const std::size_t TupleSize = std::tuple_size<Tuple>::value;

    details::TupleForEachTypeHelper<TupleSize>::template exec<Tuple>(
        std::forward<TFunc>(func));
}
```
