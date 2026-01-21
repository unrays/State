# State

It's rough, not finished, but it's very promising! I'll probably finish it very soon. Oh yes, I almost forgot, it's a state machine compile time with a transition table that's not 100% complete.

```cpp
// Copyright (c) January 2026 Félix-Olivier Dumas. All rights reserved.
// Licensed under the terms described in the LICENSE file

#include <iostream>
#include <type_traits>
#include <variant>

inline constexpr char IdleStateName[] = "Idle";
inline constexpr char PausedStateName[] = "Paused";
inline constexpr char RunningStateName[] = "Running";

inline constexpr char StopEventName[] = "Stop";
inline constexpr char PauseEventName[] = "Pause";
inline constexpr char StartEventName[] = "Start";


template<const char* Name>
struct Label {
    inline static constexpr const char* toString() { return Name; }
};


struct IState {};
struct IEvent {};

struct InvalidState : IState {};
struct InvalidEvent : IEvent {};

struct Idle    : IState, Label<IdleStateName>    {};
struct Paused  : IState, Label<PausedStateName>  {};
struct Running : IState, Label<RunningStateName> {};

struct Stop  : IEvent, Label<StopEventName>  {};
struct Pause : IEvent, Label<PauseEventName> {};
struct Start : IEvent, Label<StartEventName> {};

using State = std::variant<Idle, Running, Paused>;

/*************************************************/

template<typename... Field>
struct EntryKey {};

template<typename... Field>
struct EntryValue {};

/*************************************************/

template<typename, typename>
struct TableEntry;

template<typename... KFields, typename... VFields>
struct TableEntry<EntryKey<KFields...>, EntryValue<VFields...>> {
    using key = EntryKey<KFields...>;
    using value = EntryValue<VFields...>;
};

/*************************************************/

template<typename>
struct entry_key_size_impl;

template<typename... Fields>
struct entry_key_size_impl<EntryKey<Fields...>> {
    inline static constexpr std::size_t value = sizeof...(Fields);
};

template<typename Key>
struct entry_key_size {
    inline static constexpr std::size_t value = entry_key_size_impl<Key>::value;
};

template<typename Key>
inline static constexpr std::size_t entry_key_size_v = entry_key_size<Key>::value;

/*************************************************/

template<typename>
struct entry_value_size_impl;

template<typename... Fields>
struct entry_value_size_impl<EntryValue<Fields...>> {
    inline static constexpr std::size_t value = sizeof...(Fields);
};

template<typename Value>
struct entry_value_size {
    inline static constexpr std::size_t value = entry_value_size_impl<Value>::value;
};

template<typename Value>
inline static constexpr std::size_t entry_value_size_v = entry_value_size<Value>::value;

/*************************************************/

template<typename...>
struct Table {}; //ULTRA TEMPORAIRE ULTRA TEMPORAIRE ULTRA TEMPORAIRE ULTRA TEMPORAIRE

template<
    template<template<typename...> class, template<typename...> class> class... Entries,
    template<typename...> class Key, template<typename...> class Value
>
struct Table<Entries<Key, Value>...> {};
//ici j'ai un prob, il faut que ce soit LE NOMBRE qui soit le meme, 
//faut pas que ce soit les types de Key qui soit identiques

/*************************************************/

template<typename>
struct table_count_impl;

template<typename... Entries>
struct table_count_impl<Table<Entries...>> {
    inline static constexpr std::size_t value = sizeof...(Entries);
};

template<typename Entry>
struct table_count {
    inline static constexpr std::size_t value = table_count_impl<Entry>::value;
};

template<typename Entry>
inline static constexpr std::size_t table_count_v = table_count<Entry>::value;

/*************************************************/

template<typename>
struct table_entry_size_impl;

template<typename Entry>
struct table_entry_size_impl {
    inline static constexpr std::size_t value =
        entry_key_size<typename Entry::key>::value
        + entry_value_size<typename Entry::value>::value;
};

template<typename Entry>
struct table_entry_size {
    inline static constexpr std::size_t value = table_entry_size_impl<Entry>::value;
};

template<typename Entry>
inline static constexpr std::size_t table_entry_size_v = 
    table_entry_size_impl<Entry>::value;

/*************************************************/

template<
    std::size_t N, typename Table,
    typename Enable = std::enable_if_t<(N < table_count_v<Table>)> //fragile
>
struct at_impl;

template<template<typename...> class L, typename T, typename... Ts>
struct at_impl<0, L<T, Ts...>> {
    using type = T;
};

template<std::size_t N, template<typename...> class L, typename T, typename... Ts>
struct at_impl<N, L<T, Ts...>> {
    using type = typename at_impl<N - 1, L<Ts...>>::type;
};

//possiblement inclure ma lib et utiliser celle a disposition
//meme chose pour les méthodes utilitaires extra

/*************************************************/

template<typename... Ts>
struct min { // a mettre dans ma lib
    inline static constexpr std::size_t value = //fragile et brisé je penses
        ((std::min({ table_entry_size<Ts>::value })), ...);
};//probablement preferer max pour verifier

/*************************************************/

template<typename... Ts>
struct max {
    inline static constexpr std::size_t value =
        ((std::max({ table_entry_size<Ts>::value })), ...);
};

/*************************************************/

//un peu sale mais ca fonctionne ;)

template<typename Table, typename F, std::size_t... Is>
constexpr auto
for_each_in_table_impl2(std::index_sequence<Is...>, F&& func) {
    (func(typename at_impl<Is, std::remove_cvref_t<Table>>::type{}), ...);
}

template<typename Table, typename F, std::size_t... Is>
constexpr auto
get_for_each_in_table_impl2(std::index_sequence<Is...>, F&& func) {
    return std::tuple{
        func(typename at_impl<Is, std::remove_cvref_t<Table>>::type{})...
    };
}


template<typename F, typename Table>
constexpr decltype(auto)
for_each_in_table(F&& func, Table&& table) {
    for_each_in_table_impl2<Table>(
        std::make_index_sequence<
            table_count_v<std::remove_cvref_t<decltype(table)>>
        >{},
        func
    );
}

template<typename F, typename Table>
constexpr decltype(auto)
get_for_each_in_table(F&& func, Table&& table) {
    return get_for_each_in_table_impl2<Table>(
        std::make_index_sequence<
            table_count_v<std::remove_cvref_t<decltype(table)>>
        >{},
        func
    );
}



/*************************************************/

template<typename Table>
struct TableLookup {
    inline static constexpr std::size_t size = table_count<Table>::value;

    //template<typename... Keys>
    //auto find_by_key(Keys&&...) const
    //    -> std::enable_if_t<
    //        (sizeof...(Keys) == std::)
    //    >



        //utiliser index sequence + at

        //size
        //find (key -> value)
        //contains (value -> exists)
        //potentiellement un at (index -> value)
};

/*************************************************/

//template<typename...>
//struct Entry;
//
//template<typename Current, typename Event, typename Result>
//struct Entry<Current, Event, Result> {
//    using current = Current;
//    using event = Event;
//    using result = Result;
//};

/*************************************************/

//template<typename>
//struct Table;
//
//template<template<typename...> class... Entries, typename... Fields>
//struct Table<Entries<Fields...>...> {};

/*************************************************/

//template<typename>
//struct entry_count_impl;
//
//template<typename... Fields>
//struct entry_count_impl<Entry<Fields...>> {
//    inline static constexpr std::size_t value = sizeof...(Fields);
//};
//
//template<typename Entry>
//struct entry_count {
//    inline static constexpr std::size_t value = entry_count_impl<Entry>::value;
//};
//
//template<typename Entry>
//inline static constexpr std::size_t entry_count_v = entry_count<Entry>::value;

/*************************************************/


/*************************************************/




/*************************************************/



/*************************************************/

//a la longue, faire un system de dictionnaire compile-time avec clé valeur

template<typename Current, typename Event, typename Result>
struct Transition {
    using current = Current;
    using event = Event;
    using result = Result;
};

template<typename... Transitions>
struct TransitionTable {};


template<typename Table>
struct TransitionTableDispatcher {};

template<typename... Entries>
struct TransitionTableDispatcher<TransitionTable<Entries...>> {
    constexpr void test() { //peut etre faire un truc avec lambda
        ((std::cout
            << typename Entries::current::toString() << " + "
            << typename Entries::event::toString()   << " -> "
            << typename Entries::result::toString()  << "\n"
        ), ...);

        //faire une fonction genre check_table ou something like that.
        //fait juste iterer et si A et B == true alors on return C
    }
};

//faire facade au pire pour garder le format (prefere lui)
//ou bien faire un type entry spécial pour transition
using MyFsm = TransitionTable<
    Transition<Idle, Start, Running>,
    Transition<Running, Pause, Paused>,
    Transition<Running, Stop, Idle>,
    Transition<Paused, Start, Running>,
    Transition<Paused, Stop, Idle>
>;




/*************************************************/


template<typename State, typename Event>
struct transition_impl {
    using to = State; // ou invalid state et check après
};

template<>
struct transition_impl<Idle, Start> {
    using to = Running;
};

template<>
struct transition_impl<Running, Pause> {
    using to = Paused;
};

template<>
struct transition_impl<Running, Stop> {
    using to = Idle;
};

template<>
struct transition_impl<Paused, Start> {
    using to = Running;
};

template<>
struct transition_impl<Paused, Stop> {
    using to = Idle;
};

template<typename E>
State transition(const State& state, E) {
    return std::visit([&]<typename Cur>(Cur&&) -> State {
        return typename transition_impl<
            std::remove_cvref_t<Cur>,
            std::remove_cvref_t<E>
        >::to{};
    }, state);
}

struct Engine final {
public:
    explicit Engine(State init_state = Idle{})
        : state_(std::move(init_state)) {}

public:
    template<typename Event>
    void process_event() {
        state_ = transition(state_, Event{});

        printState(state_);
    }

    void printState(const State& s) {
        std::visit([](auto&& val) {
            std::cout << "State: " << val.toString() << "\n";
        }, s);
    }


private:
    State state_;
    

};

struct Derived {
public:


private:


};

int main() {
    //using n = transition<State::Idle, Event::Start>::to;

    //std::cout << typeid(n).name() << "\n";

    using f = transition_impl<Idle, Start>::to;
    using f2 = transition_impl<f, Stop>::to;


    f result;
    f2 result2;

    //std::cout << result.toString() << "\n";
    //std::cout << result2.toString() << "\n";

    Engine e{};

    e.process_event<Start>();
    e.process_event<Pause>();
    e.process_event<Pause>();
    e.process_event<Stop>();
    e.process_event<Pause>();


    auto d = TransitionTableDispatcher<MyFsm>{};

    d.test();
    /*
        Engine e;
        e.process_event<Start>();   // Idle → Running
        e.process_event<Pause>();   // Running → Paused
        e.process_event<Start>();   // Paused → Running
        e.process_event<Stop>();    // Running → Idle
    */   //Transition<Idle, Start, Running>,  Transition<Running, Pause, Paused>,

    using Table0 = Table<
        TableEntry<EntryKey<Idle, Start>, EntryValue<Running>>,
        TableEntry<EntryKey<Running, Pause>, EntryValue<Paused>>
    >;

    using res0 = at_impl<0, Table0>::type;
    using res1 = at_impl<1, Table0>::type;

    std::cout << "Key: " << typeid(typename res0::key).name() << "\n";
    std::cout << "Value: " << typeid(typename res0::value).name() << "\n";

    constexpr std::size_t size = min<res0, res1>::value;

    std::cout << size << "\n";


    auto table = Table0{};

    auto t2 = get_for_each_in_table([](auto&&... entry) {
        return ((table_entry_size_v<std::remove_cvref_t<decltype(entry)>>), ...);
    }, table);

    std::apply([](auto&&... sz) {
        std::cout << "MIN SIZE: " << std::min({ sz... }) << "\n";
        std::cout << "MAX SIZE: " << std::max({ sz... }) << "\n";
    }, t2);

    for_each_in_table([](auto&&... entry) {
        ((std::cout << "for_each_in_table() -> " << typeid(decltype(entry)).name() << "\n"), ...);
    }, table);
}
```
