---
title: View Models in React Inspired by SwiftUI
date: "2021-01-23T04:26:44.232Z"
description: "An elegant way to define your view models."
---

[mvvm]: https://en.wikipedia.org/wiki/Model–view–viewmodel
[camel case]: https://en.wikipedia.org/wiki/Camel_case
[email]: mailto:grepug@icloud.com
[twitter]: https://twitter.com/grepug

As a Web / JavaScript / React Developer for years, I have to admit that I hardly understand the term [MVVM], which short for Model–view–viewmodel. My code typically looked like this back then.

```tsx
interface ServerDefinedUserInfo {
  id: string
  user_name: string
  avatar_url: string
}

interface ServerDefinedItem {
  id: string
  user_info: ServerDefinedUserInfo
  // in
  date: string
  title: string
  content: string
}

function List() {
  const [list, setList] = useState<ServerDefinedItem[]>([])

  useEffect(() => {
    let didCancel = false

    async function request() {
      const res: ServerDefinedItem[] = await fetch("/my-server-endpoint")

      if (!didCancel) {
        setList(res)
      }
    }

    request()

    return () => {
      didCancel = true
    }
  }, [])

  return (
    <ListContainer>
      {list.map(item => (
        <Item key={item.id} item={item} />
      ))}
    </ListContainer>
  )
}

function Item(props: { item: item }) {
  return (
    <div>
      <User userInfo={item.user_info} />
      <Title>{item.title}</Title>
      <Content>{item.content}</Content>
      <caption>{item.date}</caption>
    </div>
  )
}
```

It may be a common pattern that first makes an HTTP request to the server, gets the data, and uses that data format direct in our views. It is OK that the server returned data fortunately fits our views. But what if

- our Views (React components) only accept props that the keys are in [camel case]
- we should format the date property from ISO format to a user readable format
- the first letter of `title` needs to be capitalized

We can solve these problems by writing some transformative code right in the `Item` component like:

```tsx
function Item(props: { item: item }) {
  const title = item.title[0].toUpperCase() + item.title.slice(1)

  const date = new Date(item.date)
  const formattedDate = `${date.getMonth() + 1}/${date.getDate()}`

  return (
    <div>
      <User userInfo={item.user_info} />
      <Title>{title}</Title>
      <Content>{item.content}</Content>
      <caption>{formattedDate}</caption>
    </div>
  )
}
```

However, things could get easily messy if we have to write more code to transform the data format from the server to meet the requirement of our Views (React components). That could be a disaster for the readability and maintainability of the code.

And this is where View Model comes into play.

## View Models

View models are like a middle layer between the models (such as data from the server) and the views. They take any data source, such as data from the server, localStorage, or memory cache, as the input, and simply output the properties directly to fit your views. Basically, view models are defined by `class`, so you can write any custom methods to transform your data as well as give any functionalities accociate with them.

Let's take a look.

```tsx
class ItemViewModel {
  readonly id: string
  readonly userInfo: UserInfoViewModel
  readonly title: string
  readonly ISODate: string

  static fromServer(item: ServerDefinedItem): ItemViewModel {
    return new ItemViewModel({
      id: item.id,
      userInfo: UserInfoModel.fromServer(item.user_info),
      title: item.title,
      ISODate: item.date,
    })
  }

  constructor(props?: Partial<ItemViewModel>) {
    Object.assign(this, props)
  }

  get firstLetterCapitalizedTitle() {
    return this.title[0] + this.title.slice(1)
  }

  get formattedDate() {
    const date = new Date(this.ISODate)
    return `${date.getMonth() + 1}/${date.getDate()}`
  }
}
```

```tsx
class UserInfoViewModel {
  readonly id: string
  readonly userName: string
  readonly avatarURL: string

  static fromServer(item: ServerDefinedUserInfo) {
    return new UserInfoViewModel({
      id: item.id,
      userName: item.user_name,
      avatarURL: item.avatar_url,
    })
  }

  constructor(props?: Partial<UserInfoViewModel>) {
    Object.assign(this, props)
  }
}
```

As the code demonstrated, we implemented a `static fromServer(item: ServerDefinedItem): ViewModel` for each view model class as the constructor (initializer) to create the view model from the server data. Within it, we transform the server data into the view model defined format. If some nested objects encountered, like the `userInfo`, we usually also create view models for them.

Now, in our views, we just use these view models:

```tsx
function List() {
  const [list, setList] = useState<ItemViewModel[]>([])

  useEffect(() => {
    // ...

    async function request() {
      const res: ServerDefinedItem[] = await fetch("/my-server-endpoint")

      if (!didCancel) {
        setList(res.map(ItemViewModel.fromServer))
      }
    }

    request()

    // ...
  }, [])

  return (
    <ListContainer>
      {list.map(item => (
        <Item key={item.id} item={item} />
      ))}
    </ListContainer>
  )
}

function Item(props: { item: ItemViewModel }) {
  return (
    <ItemContainer>
      <User userInfo={item.userInfo} />
      <Title>{item.firstLetterCapitalizedTitle}</Title>
      <Content>{item.content}</Content>
      <caption>{item.formattedDate}</caption>
    </ItemContainer>
  )
}
```

### Immutability

You may notice that I marked every property in the view models with a `readonly`. Well, this has something to do with the approach of React to determine whether it should re-render the view. We know that, after calling `setState(newValue)`, React will compare if the previous value of the state is identical to the `newValue`, triggering a re-render if they are not identical.

However, objects including class instances in JavaScript are _referce types_, meaning that if you do this:

```tsx
setState(object => {
  object.a = 2
  return object
})
```

in fact, it won't make a _new Value_ for React in aspect of the shallow comparing, because reassigning a new property value to the `object`, wont't change the reference of the `object`. Therefore, when it comes to compare the previous and the new value of the `object`, it turns out to compare the **same reference**, which is always identical. Obviously, React won't consider the state is actually changed, following no re-render occurs.

Given this, we mark each property with `readonly`, just preventing us from accidentally changing the properties of the view model's instance. So how can we update our view models elegantly?

We can simply implement a `set(props: Partial<ViewModel>): ViewModel` method to create a new instance of the view model:

```tsx
class ItemViewModel {
  // ...
  set(props: Partial<ItemViewModel>): ItemViewModel {
    return new ItemViewModel({ ...this, ...props })
  }
  // ...
}
```

Together with `setState(viewModel => ViewModel)`, we can:

```tsx
setState(item => item.set({ title: "New Title" }))
```

Now, each call to set state, React can clearly figure out that our state is definitely changed, because the reference of the previous state and the new state are totally different.

## Useful Helpers

With the powerful custom React hooks and generics from TypeScript, we can build some useful helper hooks to handle our common operations.

#### `useImmutableState<State>(initialState: State)`

```tsx
interface Immutable<T> {
  set: (props: Partial<T>) => T
}

function useImmutableState<T extends Immutable<T> | undefined>(
  initialState: T | (() => T)
) {
  const [state, _setState] = useState(initialState)

  function setState(props: Partial<T> | ((s: T) => T)) {
    if (typeof props === "function") {
      _setState(props)
    } else {
      _setState(s => (s ? s.set(props) : s))
    }
  }

  return [state, setState] as [Readonly<typeof state>, typeof setState]
}
```

Note that the interface `Immutable<T>` can be used to be `implement`ed by view model classes:

```tsx
class ItemViewModel implements Immutable<ItemViewModel> {
  // ...
}
```

Now, we have the compiler's guarantee that we did implement the `set(props: T): T` method on the view model.

With `useImmutableState()`, we now are able to use it exactly like the ordinary `useState()`, even with more type safety.

#### `useImmutableArrayState<State>(initialState: State)`

```tsx
function useImmutableArrayState<T extends Immutable<T>>(
  initialList: Readonly<Readonly<T>>[] | (() => Readonly<Readonly<T>>[])
) {
  const [list, _setList] = useState(initialList)

  function setList(index: number, el: Partial<T>) {
    _setList(list => {
      const newInstance = list[index].set(el)

      return [...list.slice(0, index), newInstance, ...list.slice(index + 1)]
    })
  }

  return [list, _setList, setList] as [
    typeof list,
    typeof _setList,
    typeof setList
  ]
}
```

In terms of array mutations, we can simply replace the item in the array that we want to change with a new created item instance.

## Conclusion

With view models introduced, we can do really bunch of opearations in a elegant way right in the view model's class definitions. There are way more usage to explore, for example

- implementing a `toJSON()` method, we can customize the behavior of `JSON.stringify()` method call on the instance of view model.
- implementing a `static fromCache(): ViewModal` method, we can restore the state of the view model from any types of cache.
- implementing everything related to CRUD operations.

More than that, like I said above, we can use any data source to construct our view models. So, besides server data, we can mock our data in the development stage by simply implementing a `static fromMock(): ViewModel` method.

### Inspired by Swift / SwiftUI

In the main title I said that this idea is inspired by SwiftUI. I want to tell some story about it.

With the introduction of SwiftUI, and finding that the main idea behind SwiftUI is way similiar to React, I decided to learn something about it. Ironically, I had started to review the idea of MVVM in the process of learning SwiftUI. And I tried to apply that idea in the React app development. Only I found was that there is no `struct`-like _value type_ in native JavaScript, every object is a _reference type_. So I tried to understand the _value type_ in Swift, and came up with an idea that creating a new reference on each view model update could simulate the value type's behavior in JavaScript.

Hope this article can be helpful for you. Feel free to reach out via either [twitter] or [email].
