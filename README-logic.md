The principal operation of a website is to stream data between endpoints. Bytes start at the user's keyboard, pass through layers of application logic, land in the database and return back as impeccable HTML and CSS constructions.

`kea-logic` makes these transformations a joy to perform on the frontend. It uses components you know and love (`react`, `redux`, `redux-saga`, `reselect`, `react-router`) to create a well-oiled machine that brings your data to life.

There are many other projects under the [`kea`](https://github.com/mariusandra/kea) name that you may want to check out.

# kea-logic

`kea-logic` gives you logic stores with actions. These act as glue between React components and sagas.

The logic stores are designed to be well-readable, self-documenting and easily refactorable. You'll know how it works even if you have never seen the code before.

In addition to this, `kea-logic` provides helpers to simplify routing, code splitting and other parts of a good webapp.

## Try it out!

Open the [demo app](http://example.kea.rocks/), [browse its code](https://github.com/mariusandra/kea-example) and read below for an explanation of the parts.

To run the example on your machine, just type these commands:

```
npm install kea -g
kea new my-project
cd my-project
npm start
```

and open [http://localhost:2000/](http://localhost:2000/).

Later inside `my-project` run these to hack away:

```
kea g scene-name                               # new scene
kea g scene-name/component-name                # component under the scene
kea g scene-name/component-name/really-nested  # deeply nested logic
```

# Logic stores

Logic stores consist of 1) actions, 2) structure (reducer + selector + proptype), and 3) optionally sagas.

Logic stores are connected through ES6 imports.

Let's first look at a React component that uses these stores:

```js
// scenes/homepage/index.js - This the root component for the homepage scene. skipping some imports

// A helper component.
import Slider from '~/scenes/homepage/slider'

// Note, you should always import with the full path, so you can easily move things around
// and refactor just by searching for the path.

// logic stores: 1) for this "homepage" scene root component and 2) the slider helper component
import sceneLogic from '~/scenes/homepage/logic'
import sliderLogic from '~/scenes/homepage/slider/logic'

// select which fields of data and which actions we want from the above imported logic stores
const mapping = {
  actions: [
    sceneLogic, [
      'updateName'
    ]
  ],
  props: [
    sceneLogic, [
      'name',
      'capitalizedName'
    ],
    sliderLogic, [
      'currentSlide',
      'currentImage'
    ]
  ]
}

class HomepageScene extends Component {

  // react will know the PropTypes automatically
  static propTypes = propTypesFromMapping(mapping, { /* extra PropTypes if needed */ })

  // binding to 'this', hence the fat arrow syntax
  // this way we can just pass onClick={this.updateName} in render()
  updateName = () => {
    // each function defines on top which props and actions it needs
    const { name } = this.props
    const { updateName } = this.props.actions

    const newName = window.prompt('Please enter the name', name)

    if (newName) {
      updateName(newName) // call the action to update the data
    }
  }

  // render the component
  render () {
    // the data we need from the imported stores
    const { capitalizedName, currentSlide, currentImage } = this.props

    return (
      <div className='homepage-scene'>
        <Slider />
        <h1>
          Hello, I am <em onClick={this.updateName}>{capitalizedName}</em> the Kea
        </h1>
        <p>
          You are viewing image #{currentSlide + 1}, taken by <a href={currentImage.url}>{currentImage.author}</a>
        </p>
      </div>
    )
  }
}

// finally, connect the mapping to the scene
export default connectMapping(mapping)(HomepageScene)
```

Logic stores consist of many parts:

They all have a path in the redux tree.

```js
// scenes/homepage/logic.js
class SceneLogic extends Logic {
  // PATH
  path = () => ['scenes', 'homepage', 'index']
```

All the data declared below will live in your redux store under `scenes.homepage.index`.

The action generators are inspired by `redux-act`, but don't need a description. Give them a keyword and the payload object generator.

```js
  // ACTIONS
  actions = ({ constants }) => ({
    updateName: (name) => ({ name }),
    increaseAge: (amount = 1) => ({ amount }),
    decreaseAge: (amount = 1) => ({ amount })
  })

  // Calling `updateName('new name')` will create an object like:
  // { type: 'updateName@homepage.index', payload: { name: 'new name' } }
```

Logic stores have a *structure*, created with `redux` and `reselect` plus type declarations and optional persistence. Everything here is a pure function working with immutable data.

```js
  // STRUCTURE
  structure = ({ actions, constants }) => ({
    name: createMapping({
      [actions.updateName]: (state, payload) => payload.name
    }, 'Chirpy', PropTypes.string),

    age: createMapping({
      [actions.increaseAge]: (state, payload) => state + payload.amount,
      [actions.decreaseAge]: (state, payload) => Math.max(state - payload.amount, 1)
    }, 3, PropTypes.number, { persist: true })
  })
```

Finally, we add custom [`reselect`](https://github.com/reactjs/reselect) selectors to transform and cache data:

```js
  // SELECTORS
  selectors = ({ path, structure, constants, selectors, addSelector }) => {
    // define the name of the selector, its PropType and tell it what data you want
    addSelector('capitalizedName', PropTypes.string, [
      selectors.name,
    ], (name) => {
      return name.trim().split(' ').map(k => `${k.charAt(0).toUpperCase()}${k.slice(1).toLowerCase()}`).join(' ')
    })

    addSelector('description', PropTypes.string, [
      selectors.capitalizedName,
      selectors.age
    ], (capitalizedName, age) => {
      return `Hello, I'm ${capitalizedName}, a ${age} years old bird!`
    })
  }
}
```

Logic stores are exported as singletons from these `logic.js` files:

```js
export default new SceneLogic().init()
```

Check out the [TodoMVC logic.js](https://github.com/mariusandra/kea-example/blob/master/app/scenes/todos/logic.js) for a more complete example.

Once defined, a logic store can be imported in any component, saga or other logic store as needed.

```js
import sceneLogic from '~/scenes/homepage/logic'

// use them just like this
sceneLogic.path === ['scenes', 'homepage', 'index']
sceneLogic.actions === { updateName: function(name), increaseAge: function(amount), ... }
sceneLogic.reducer === function (state, action) { ... }
sceneLogic.selector === (state) => state.scenes.homepage.index
sceneLogic.selectors === { name: (state) => state.scenes.homepage.index.name, ... }

// or plug them into other kea-logic components for maximum interoperability
```

# Side effects (API calls, etc)

Since logic stores are strictly composed of [pure functions](https://en.wikipedia.org/wiki/Pure_function) that operate on [immutable data](https://en.wikipedia.org/wiki/Immutable_object), we need an alternative place to handle all of the messy business logic of the app.

Enter [`sagas`](https://github.com/yelouafi/redux-saga).

This is where you wait for actions to be triggered, run async processing logic, and send the results back through another action.

Using sagas complex async processing logic can be written in an elegant and linear fashion. Here's one example that updates the homepage slider component:

```js
// scenes/homepage/slider/saga.js
import delay from '~/utils/delay'

import sliderLogic from '~/scenes/homepage/slider/logic'

// we want to call the updateSlide action on the slider's logic store
const actions = selectActionsFromLogic([
  sliderLogic, [
    'updateSlide'
  ]
])

export default function * saga () {
  const { updateSlide } = actions

  while (true) {
    // wait for someone to call the updateSlide action or 5 seconds to pass
    const { change, timeout } = yield race({
      change: take(updateSlide().type),
      timeout: delay(5000)
    })

    // if timed out, advance the slide
    if (timeout) {
      const currentSlide = yield sliderLogic.get('currentSlide')
      yield put(updateSlide(currentSlide + 1))
    }

    // reset the clock to 5 seconds and wait again
  }
}
```

Read the documentation for [`redux-saga`](https://github.com/yelouafi/redux-saga) to grasp their power!

# Scenes

You can always treat the logic store reducers and sagas manually and plug them into your existing application.

If, however, you favor convenience, you may combine them into scenes.

Scenes are defined in `scene.js` files like so:

```js
// scenes/homepage/scene.js
import { createScene } from 'kea-logic'

import sceneComponent from '~/scenes/homepage/index'
import sceneLogic from '~/scenes/homepage/logic'
import sceneSaga from '~/scenes/homepage/saga'

import sliderLogic from '~/scenes/homepage/slider/logic'
import sliderSaga from '~/scenes/homepage/slider/saga'

export default createScene({
  name: 'homepage',
  component: sceneComponent,
  logic: [
    sceneLogic,
    sliderLogic
  ],
  sagas: [
    sceneSaga,
    sliderSaga
  ]
})
```

You may then access the combined scene like so:

```js
import homepageScene from '~/scenes/homepage'

homepageScene.saga === function * () { ... }                    // start the scene sagas in parallel
homepageScene.combineReducers() === function (state, action) {} // calls redux's combineReducers
```

or plug it into the kea-logic routing helpers.

# Routes

Give `redux-router` a helping hand:

```js
// routes.js
import { combineScenesAndRoutes } from 'kea-logic'

const scenes = {
  homepage: require('bundle?lazy&name=homepage!./homepage/scene.js'),
  todos: require('bundle?lazy&name=todos!./todos/scene.js')
}

const routes = {
  '/': 'homepage',
  '/todos': 'todos',
  '/todos/:visible': 'todos'
}

export default combineScenesAndRoutes(scenes, routes)
```

... and have those scenes lazily loaded when route is accessed.

# Code structure

While logic stores can exist anywhere, it is highly recommended to organise your code like this:

* `scenes` - a scene is a page or a subsystem in your app
* `components` - react components that are shared between scenes
* `utils` - javascript utils shared between scenes

Side note: as we strive for simplicity, readability and clarity, we will use [JavaScript Standard Style](http://standardjs.com/rules.html) and skip semicolons. They are added/removed as needed in the transpiling/minimising stage, and add no value. Any "I forgot the semicolon" errors you might be worried about will be caught by the linter anyway. (Please install eslint and plugins for your IDE!)

Here's a typical structure:

```
scenes/homepage/
- index.js    # the react component
- logic.js    # actions, reducers, selectors
- saga.js     # saga
- styles.scss # styles for this scene
- scene.js    # one of these per scene

scenes/homepage/slider/
- _assets/    # some images
- index.js    # the react component
- logic.js    # actions, reducers, selectors
- saga.js     # saga
- styles.scss # styles for the slider

scenes/todos/
- index.js    # the react component
- logic.js    # actions, reducers, selectors
- saga.js     # saga
- styles.scss # styles for this scene
- scene.js    # one of these per scene

scenes/todos/todo/
- index.js    # the react component

components/
- header/
  - index.js
  - styles.scss

utils/
- create-uuid.js
- range.js
- delay.js

scenes/
- index.js
- routes.js
- styles.scss

/
- index.html
- index.js
- store.js
```

## What is `kea`?

All of this is a work in progress. Any and all help is greatly appreciated!

Read more here about the quest to find [the smartest way to develop react applications](https://github.com/mariusandra/kea).

## List of public functions:

```js
export default Logic
export { pathSelector, createSelectors } from './selectors'
export { selectPropsFromLogic, propTypesFromMapping, havePropsChanged } from './props'
export { selectActionsFromLogic } from './actions'
export { createPersistentReducer, createStructureReducer } from './reducer'
export { createCombinedSaga } from './saga'
export { createScene } from './scene'
export { createMapping } from './structure'
export { connectMapping } from './connect'
export { getRoutes, combineScenesAndRoutes } from './routes'
export { NEW_SCENE, createRootSaga, createKeaStore } from './store'
```

More documentation coming soon! Please help if you can!