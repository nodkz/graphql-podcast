# Paypal

## GraphQL: A success story for PayPal Checkout

Mark Stuart Oct 17, 2018

https://medium.com/paypal-engineering/graphql-a-success-story-for-paypal-checkout-3482f724fb53

REST API vs GraphQL at Paypal
“... we found that UI developers were spending less than 1/3 of their time actually building UI. The rest of that time spent was figuring out where and how to fetch data, filtering/mapping over that data…”  —  @mark_stuart

https://twitter.com/nodkz/status/1195588179606290432

## Scaling GraphQL at PayPal

Mark Stuart Oct 30, 2019

https://medium.com/paypal-engineering/scaling-graphql-at-paypal-b5b5ac098810

A little bit about GraphQL performance:
Scaling people, tooling and processes are most challenging than horizontal scaling or paying a lot of money for servers or cloud computing.
Crystal thoughts and writing by @mark_stuart

https://twitter.com/nodkz/status/1195592533289684992

## PayPal && GraphQL Summit

### BUILDING A FASTER CHECKOUT EXPERIENCE AT PAYPAL WITH GRAPHQL

At PayPal, we are rebuilding our checkout experience from the ground up. The most important and logical problem we are solving is to make the experience super fast. Come hear our story on how we leveraged GraphQL server to efficiently present varied regional payment experiences to people all around the globe.

VISHAKHA SINGH
Senior Software Engineer at PayPal

https://www.youtube.com/watch?v=JneBBbDWiVs

- получить всю информацию по чекауту за 200ms
  - 12 платежных методов
  - инфу можно запрашивать параллельно
  - клиенты могут запрашивать только ту информацию, которая им нужна и не ждать генерацию ответа для ненужных способов оплат
  - каждый платежный метод изолирован и поддерживается своими командами
  - можно легко добавлять новые платежные методы без реинтеграции с  магазинами
  - резолвер это функция, поэтому можно использовать memoization
  - для некоторых API мы кешируем ответ, чтоб лишний раз не дергать редко меняющиеся данные
- туллинг и инструменты
  - а вы знаете сколько стоит времени получения данных по тому или иному полю? Мы используем Аполло трейсинг.
  - замеряем кол-во вызовов по резолверам
  - замеряем время выполнения резолвера
  
### STATE MANAGEMENT IN GRAPHQL USING REACT HOOKS & APOLLO

This talks walks through how to use React Hooks in maintaining state while using GraphQL.

SHRUTI KAPOOR https://twitter.com/Shrutikapoor08
Software Engineer at PayPal

https://www.youtube.com/watch?v=Y3jXNMhAy0U

- разные типы состояния
  - глобальный стейт
    - аппликейшн стейт
    - данные доступные всем компонентам
    - данные роутинга
    - авторизация
  - локальный стейт
    - данные конкретной компоненты
    - данные форм
  - как управлять стейтом
    - React Component State
    - ContextAPI
    - Redux
    - Mobx
    - mobx-state-tree
    - Hooks
    - Apollo Client
- хуки 101
  - proposed 2018 октября
  - released 2019 февраль/март
  - stable in 16.8
  - useState – replace this.state
  - useEffect – update DOM, similar to componentDidMount, subscribe/unsubscribe
  - useContext – replacement for Context.Consumer
  - useReducer – dispatch an action to change the state (for Redux lovers)
  - useCallback – мемоизированный колбэк
  - useMemo – мемоизированное значение
- реакт хуки и Графкуэль
  - демо приложение в двух ветках custom-hooks & apollo-hooks https://github.com/shrutikapoor08/graphqlsummit2019
- аполло и хуки
  - useQuery
  - useMutation

### I HEART GRAPHQL (AND KOTLIN)

In this talk, you'll hear from a Kotlin fanboy who wants to show you how he built his GraphQL APIs.

ROHIT BASU https://twitter.com/rohitdbasu
Architect at PayPal

https://www.youtube.com/watch?v=qOFUYxinkbk

- Как поднять GraphQL сервер на Kotlin
- Почему Котлин
  - выразительный, четкий (concise)
  - совместимость (interoperable)
  - null safety
  - расширения (extensions)
  - coroutines
- graphql-kotlin-spring-server
  - автоматизирована процедура настройки через spring boot
  - в pom.xml добавляем один депенденси и Графкуэль-сервер готов
  - потом пишем класс украшаем декораторами (аннотация) и вот готов тип для схемы
- Архитектор от бога, что сказать 😅
