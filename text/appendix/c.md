# Appendix C - tupleForEachFromUntil

Implementation of `tupleAccumulate()` function. Namespace `details` contains some
helper classes.
```cpp
namespace details
{

template <std::size_t TRem, std::size_t TOff = 0>
class TupleForEachFromUntilHelper
{
public:
    template <typename TTuple, typename TFunc>
    static void exec(TTuple&& tuple, TFunc&& func)
    {
        using Tuple = typename std::decay<TTuple>::type;
        static const std::size_t TupleSize = std::tuple_size<Tuple>::value;
        static const std::size_t OffsetedRem = TRem + TOff;
        static_assert(OffsetedRem <= TupleSize, "Incorrect parameters");

        static const std::size_t Idx = TupleSize - OffsetedRem;
        func(std::get<Idx>(std::forward<TTuple>(tuple)));
        TupleForEachFromUntilHelper<TRem - 1, TOff>::exec(
            std::forward<TTuple>(tuple),
            std::forward<TFunc>(func));
    }
};

template <std::size_t TOff>
class TupleForEachFromUntilHelper<0, TOff>
{
public:
    template <typename TTuple, typename TFunc>
    static void exec(TTuple&& tuple, TFunc&& func)
    {
        static_cast<void>(tuple);
        static_cast<void>(func);
    }
};

}  // namespace details

// Invoke provided functor for every element in the tuple which indices
//     are in range [TFromIdx, TUntilIdx).
template <std::size_t TFromIdx, std::size_t TUntilIdx, typename TTuple, typename TFunc>
void tupleForEachFromUntil(TTuple&& tuple, TFunc&& func)
{
    using Tuple = typename std::decay<TTuple>::type;
    static const std::size_t TupleSize = std::tuple_size<Tuple>::value;
    static_assert(TFromIdx < TupleSize,
        "The from index is too big.");

    static_assert(TUntilIdx <= TupleSize,
        "The until index is too big.");

    static_assert(TFromIdx < TUntilIdx,
        "The from index must be less than until index.");

    static const std::size_t FieldsCount = TUntilIdx - TFromIdx;

    details::TupleForEachFromUntilHelper<FieldsCount, TupleSize - TUntilIdx>::exec(
        std::forward<TTuple>(tuple),
        std::forward<TFunc>(func));
}

```
