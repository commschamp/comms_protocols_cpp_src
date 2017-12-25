# Appendix A - tupleForEach

Implementation of `tupleForEach()` function. Namespace `details` contains some
helper classes.
```cpp
namespace details
{
template <std::size_t TRem>
class TupleForEachHelper
{
public:
    template <typename TTuple, typename TFunc>
    static void exec(TTuple&& tuple, TFunc&& func)
    {
        using Tuple = typename std::decay<TTuple>::type;
        static const std::size_t TupleSize = std::tuple_size<Tuple>::value;
        static_assert(TRem <= TupleSize, "Incorrect parameters");

        // Invoke function with current element
        static const std::size_t Idx = TupleSize - TRem;
        func(std::get<Idx>(std::forward<TTuple>(tuple)));
        
        // Compile time recursion - invoke function with the remaining elements
        TupleForEachHelper<TRem - 1>::exec(
            std::forward<TTuple>(tuple),
            std::forward<TFunc>(func));
    }
};

template <>
class TupleForEachHelper<0>
{
public:
    // Stop compile time recursion
    template <typename TTuple, typename TFunc>
    static void exec(TTuple&& tuple, TFunc&& func)
    {
        static_cast<void>(tuple);
        static_cast<void>(func);
    }
};
}  // namespace details

template <typename TTuple, typename TFunc>
void tupleForEach(TTuple&& tuple, TFunc&& func)
{
    using Tuple = typename std::decay<TTuple>::type;
    static const std::size_t TupleSize = std::tuple_size<Tuple>::value;

    details::TupleForEachHelper<TupleSize>::exec(
        std::forward<TTuple>(tuple),
        std::forward<TFunc>(func));
}
```
