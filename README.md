# Realm JS Interface Proposal

## Summary
We are in discussions on how to best update our API so that it is more ergonomic for modern React Native development.  Many of the ideas presented here are theoretical and may be subject to change based on the feasibility and limits of React Native and Javascript.  The following examples are written in Typescript, but Javascript will continued to be supported.
## Model Definition

The following is using decorators, which is not yet part of the JavaScript standard.  Enabling this would require small tweaks to [babel](https://babeljs.io/docs/en/babel-plugin-proposal-decorators) or the [typescript tsconfig](https://www.typescriptlang.org/docs/handbook/decorators.html)

It is desired to reduce the boilerplate code of writing a schema and reduce this into a class definition with options provided in decorator functions.  In this example, the Realm property types would be infered from the given Typescript type.

```typescript
import Realm, {ObjectId, List, Results} from 'realm';
import {model, property, linkedTo} from '@realm/decorators';

@model()
export class TodoList {
    @property({primary: true}) _id: ObjectId;
    @property() name: String;
    @property() items: List<TodoItemSchema>;

    constructor(name: string) {
        this._id = new ObjectId();
        this.name = name;
        this.items = new List<TodoItem>();
    }
}

@model()
export class TodoItem {
    @property({primary: true, type: 'objectId'}) _id: ObjectId;
    @property({type: 'string'}) description: String;
    @property({default: false}) done?: Boolean;
    @property({type: 'date'}) deadline?: Date;
    @linkedFrom(TodoList, 'items') lists: Results<TodoListSchema>;

    constructor(description: string) {
        this._id = new ObjectId();
        this.description = description;
    }


    // The mutator decorator will automatically wrap this operation in a write transaction
    @mutator toggleDone(){
        this.done = !this.done
        this.dateDone = new Date()
    }
}

// Path will be required for multiple realms
export const TodoRealm = createRealmProvider({schemas=[TodoItem, TodoList], path: "whatever.realm"})
```

## Project Setup

Setting up realm should be easy as wrapping the application with a provider.  This allows access to the realm object anywhere in the app tree.

```typescript
import { RealmProvider } from "realm"
import { TodoItem, TodoList, TodoRealm } from "./models"
  
export const Main = () => {
    return (
        <TodoRealm.Provider>
              <App/>
        </TodoRealm.Provider>
    )
}
```


## Realm Components

```tsx
const TodoListView = (listId: ObjectId) => {
    // Get object by primary key
    const todoList = TodoRealm.useObject(TodoList, listId)

    if(!todoList){
        return null;
    }

    return (
        <FlatList
            data={todoList.items}
            renderItem={({item}) => {
                return <TodoItemView item={item} />;
            }}
            keyExtractor={item => `${item._id}`}
        />
    )
}

const TodoListView = (item: TodoItem) => {
    return (
        <>
            <Text>{todoItem.description}</Text>
            <Touchable
                onPress={() => {
                    todoItem.toggleDone()
                }}>
                <View style={styles.checkIconContainer}>
                    {todoItem.done ? (
                        <Icon name="checked"/>
                    ) : (
                        <Icon name="circle"/>
                    )}
                </View>
            </Touchable>
        </>
    )
}
```

### List of Objects View
```tsx
const TodoListView = (listIndex: ObjectId) => {
    // Return all the lists in the database
    const todoLists = TodoRealm.useQuery(
        TodoList, {sort: "name ASC", filter: "name != main"}
    )

    return (
        <>
            {todoLists.map(item => <>{item.name}</>)}
        </>
    )
}
```

### Create

```tsx
const DraftTodoList = () => {
    const createTodoList = TodoRealm.useCreator(TodoListModel)

    const [draftTodoList, setDraftTodoList] = useState<TodoList>((name: 'Placeholder Text'))

    return (
        <>
            <TextInput
            />
            <Button 
                title={"create"}
                onPress={() => createTodoList(draftTodoList)}
            />
        </>
    )

}
```
### Destroy
```tsx
const DraftTodoList = () => {
    const destroyObject = TodoRealm.useDestroyer()

    const todoList = TodoRealm.useObject(TodoList, listIndex)

    return (
        <>
            <TextInput
            />
            <Button 
                title={"destroy"}
                onPress={() => destroyObject(todoList)}
            />
        </>
    )

}
```

### Update Transaction
```tsx
const DraftTodoList = (item:Realm.Result<TodoItem>) => {
    // draft will be a copy of the realm result as a js object
    const [draft, setDraft, commitDraft] = TodoRealm.useDraft(item)

    return (
        <>
            <TextInput
                value={draft.name}
                onTextChange={value => setDraft({...draft, name: value.name}) }
            />
            <Button 
                title={"create"}
                onPress={() => 
                    commitDraft()
                }
            />
        </>
    )

}
```

### In emergency realm is always there

`const realm = TodoRealm.useRealm()`

## TODO
* handle multiple realms
* option to define a write transaction (rollback on fail)