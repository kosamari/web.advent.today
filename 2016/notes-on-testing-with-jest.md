## Notes on testing with Jest

[Jest](http://facebook.github.io/jest) is a test runner for testing Javascript code.

A test runner is a piece of software that looks for tests on your codebase and runs them. It also takes care of displaying the result on the CLI interface. Jest boasts of having __very fast__ test execution as it runs tests in parallel.

Jest provides a variety of features optimised to run our tests in a convenient and fast manner. 

- **Easy configuration** For trivial cases, `jest` can be run without any configuration. If needed, jest provides a variety of ways to customise your tests by providing relevant parameters in `package.json` or as a separate `jest.config.js` file.

 `jest --config jest.config.js`

- **Snapshot testing** 
Much has been written about this kind of testing. Jest provides a way to take "snapshots" of your DOM components, which is essentially a file with the HTML of the component's render stored as a string. The snapshots are human readable and act as an indicator of any DOM change due to component code changes.

- **Mocking**
Jest provides easy ways to auto-mock or manually mock entities. You can mock functions and modules which are irrelevant to your test. Jest also provides fake timers for you to control the `setTimeout` family of functions in tests.

- **Interactive CLI** 
Jest has a watch mode which watched for any test or relevant code changes and re-runs the respective tests. The watch has an interactive feature to it wherein it provides options to run all tests or fix snapshots while in watch mode itself. This is incredibly convenient.

- **Coverage metrics** Jest provides some awesome coverage metrics right out of the box.

  `jest --coverage`

It can be configured to show different levels of these metrics. Also, it is possible to cap the coverage to a threshold and bail or error out if the threshold is breached. Your tests, however will complete very slowly if this feature is enabled.

In this post, we will look at installing `jest` and getting started with DOM testing a `react` application. Theoretically these techniques could be used with other view frameworks like `deku`, `preact` and so on. But there may be challenges with finding proper rendering utils which are mature and feature rich.

### Setting Up Jest

Install `jest` for your project

```javascript
npm install --save-dev jest
```

Add `npm test` script

__package.json__
```javascript
{
    ...
    scripts: {
        ...
        test: 'jest --watch --verbose'
    }
    ...
}
```

### DOM Testing

In the following examples we will test mostly trivial methods of general testing areas for a front end application.
This involves
 - Basic DOM testing
 - Event simulation and testing
 - Window events simulation

The sample code can be found at [jest-blog-samples](https://github.com/pksjce/jest-blog-samples)

#### Basic DOM Test

The app itself consists of a react component `App` which prints out a `div` with text "Hello jest from react".

[__index.js__](https://github.com/pksjce/jest-blog-samples/blob/master/dom-testing/index.js)

```javascript
import React, {Component} from 'react';
import {render} from 'react-dom';

export default class App extends Component {
    render() {
        return <div>Hello jest from react</div>;
    }
}

render(<App/>, document.body)
```

The above app is bundled using [`webpack`](https://github.com/pksjce/jest-blog-samples/blob/master/dom-testing/webpack.config.js)

My first test is to check if the `App` component renders out the `div`.

[__index.test.js__](https://github.com/pksjce/jest-blog-samples/blob/master/dom-testing/__tests__/index.test.js)

```javascript
import React from 'react';
import App from '../index';
import {shallow} from 'enzyme';


describe('The main app', () => {
    it('the app should have text', () => {
        const app  = shallow(<App/>);
        expect(app.contains(<div>Hello jest from react</div>)).toBe(true);
    })
})
```

Lets break this down.

* We need to import `react`(for the purpose of using jsx) and the `App`(to instantiate and test) source.
* [`enzyme`](https://github.com/airbnb/enzyme) is JavaScript test utility that helps render `react` components for testing. We will use it to `shallow render` our `App` component and inspect the resulting tree.
* We describe a test suite named "The main app".
* Our first test is named "the app should have text"
* We [shallow render](https://github.com/airbnb/enzyme/blob/master/docs/api/shallow.md) our component and store it in a `const`.
* Add an `expect` assertion for the app to contain the expected `div`.

#### Run the test with `jest`

Assuming jest is installed locally for the application and you have configured the `test` script in `package.json` as recommended, let's first start the test runner.

```bash
npm run test 
```

This automatically finds the tests to be run. If required, one can specify [testPathDirectories](https://facebook.github.io/jest/docs/configuration.html#testpathdirs-array-string) and [testIgnorePaths](https://facebook.github.io/jest/docs/configuration.html#testpathignorepatterns-array-string).

```javascript
 FAIL  __tests__/index.test.js
  ● Test suite failed to run

    SyntaxError: /Users/pavithra.k/Workspace/jest-blog-samples/dom-testing/__tests__/index.test.js: Unexpected token (8:29)
       6 | describe('The main app', () => {
       7 |     it('the app should have text', () => {
    >  8 |         const app  = shallow(<App/>);
         |                              ^
       9 |         expect(app.contains(<div>Hello jest from react</div>)).toBe(true);
      10 |     })
      11 | })
      
      at Parser.pp.raise (node_modules/babylon/lib/index.js:4215:13)

Test Suites: 1 failed, 1 total
Tests:       0 total
Snapshots:   0 total
Time:        1.07s
Ran all test suites.
```

We have our first failing test!

The error mainly points at a syntax error. This is because are test code is wriiten in ES2015 and it is not transpiled and ready for node.

This means `jest` needs to preprocess our test code and imported source code to ES5. For this, jest has great integration with [`babel`](https://babeljs.io/)

#### Add `babel` Support

Add a [`.babelrc`](https://github.com/pksjce/jest-blog-samples/blob/master/dom-testing/.babelrc) file to your application. In this you can specify the babel plugins required to transpile every ES2015 feature that your application uses.
`babel` provides excellent presets for [`react`](https://babeljs.io/docs/plugins/preset-react/) and [`es2015`](https://babeljs.io/docs/plugins/preset-es2015/). Presets are just a collection of relevant babel plugins for a logical set of transformations.

jest provides with it an inbuilt babel preprocessor, called `babel-jest`. Essentially, this just transpiles all the tests before running them, if there is a `.babelrc` file present in the application.

Now on running `jest`, we have

```javascript
 PASS  __tests__/index.test.js
  The main app
    ✓ the app should have text (8ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        2.175s
Ran all test suites.
```

#### Test an event execution

Using `enzyme` we can simulate events in our rendered components.

We intend to build an App Component that is clickable. It should change the text displayed after the click on it's body.

To test this component we need to

 - Test for default content on first load.
 - Test for first toggle of click and expect the change in text.
 - Test for second toggle of click and again expect change in text to the default.

[__tests__/index.test.js](https://github.com/pksjce/jest-blog-samples/blob/master/event-testing/__tests__/index.test.js)

```javascript
import React from 'react';
import App, {DEFAULT_TEXT, CLICKED_TEXT} from '../index';
import {shallow} from 'enzyme';

describe('The main app', () => {
    let app;
    beforeEach(() => {
        app  = shallow(<App/>);
    })

    it('the app should have text', () => {
        expect(app.text()).toBe(DEFAULT_TEXT);
    })
    it ('should change text on click', () => {
        app.simulate('click');
        expect(app.text()).toBe(CLICKED_TEXT);
        app.simulate('click');
        expect(app.text()).toBe(DEFAULT_TEXT);
    })
})
```

`jest` provides us with the following [global testing methods](http://facebook.github.io/jest/docs/api.html)

__`describe(name,fn), it(name, fn), test(name, fn)`__

These are the basic tools in a javascript testing environment's [arsenal](https://en.wikipedia.org/wiki/Arsenal_F.C.).
`describe` is used to group a set of related tests together into a __test suite__.
`it` or `test` is used to write your actual test where you can build an independent environment and test particular units or behaviour.

__`afterEach(fn)/beforeEach(fn)/afterAll(fn)/beforeAll(fn)`__

These functions run before/after each test or before/after the test suite. You can group all your initialization and tear down code in these hooks which are called during a test cycle.

Keeping the above info in context, we write a `beforeEach` function which creates a shallow rendering of the Button `App`.
After this we write our tests for each of the behaviour we expect out of the component. Enzyme has a [bunch of helper api](http://airbnb.io/enzyme/docs/api/shallow.html) to inspect a rendered object. We can use these to check rendered DOM for desired changes.

The corresponding react component would look like this.

[__index.js__](https://github.com/pksjce/jest-blog-samples/blob/master/event-testing/index.js)

```javascript
import React, {Component} from 'react';
import {render} from 'react-dom';

export const DEFAULT_TEXT = "Hello jest from react";
export const CLICKED_TEXT = "This has been clicked";

export default class App extends Component {
    constructor() {
        super();
        this.state = {
            clicked: false
        }
        this.handleClick = this.handleClick.bind(this);
    }
    render() {
        return <div onClick={this.handleClick}>{this.state.clicked ? CLICKED_TEXT : DEFAULT_TEXT}</div>;
    }
    handleClick() {
        this.setState({
            clicked: !this.state.clicked
        })
    }
}

render(<App/>, document.body)
```

#### `window` event testing

Many times, our apps have functionality to respond to outer stimuli like `window` events. We would like to test if our app behaves correctly under these conditions.
For this case, we will have to mock the `document` and `document.window` object.

We will use [`jsdom`](https://github.com/tmpvar/jsdom#for-the-hardcore-jsdomjsdom) api which provides a way of faking `document` and `window` objects.

```bash
npm install --save-dev jsdom
```

We are now going to use `jsdom` to wire up our fake objects. Since this activity needs to happen before every test as any of them could want the window object, we will write this code at a `setupFile` which can be configured in `jest`

Go to your `package.json` or `jest.config.js` file and add the following [option](http://facebook.github.io/jest/docs/configuration.html#setupfiles-array)

[__package.json__](https://github.com/pksjce/jest-blog-samples/blob/master/window-testing/package.json)

```json
jest: {
    "setupFiles" : ["<rootDir>/setupFile.js"]
}
```

[__setupFile.js__](https://github.com/pksjce/jest-blog-samples/blob/master/window-testing/setupFile.js)

```javascript
import {jsdom} from 'jsdom';

const documentHTML = '<!doctype html><html><body><div id="root"></div></body></html>';
global.document = jsdom(documentHTML);
global.window = document.parentWindow;

global.window.resizeTo = (width, height) => {
	global.window.innerWidth = width || global.window.innerWidth;
	global.window.innerHeight = width || global.window.innerHeight;
	global.window.dispatchEvent(new Event('resize'));
};

```

Note that the above code also gives us a `div` element with id `root`. We can now `mount` our react component onto this element for inclusion in the DOM.

Also, `resizeTo` will not be defined by default in this fake `window` object. It will return `undefined`. Hence we need to mock up the whole function too. You will need to do this to trigger any event on the `window` from your test.
Currently I'm only bothered with the `innerWidth` and `innerHeight` parameters. Hence I will only update them as part of the mock.

Our App code for this test is a react component that responds to `window`'s resize event. If the resize results in the `innerWidth > 1024` then it changes its `width` to `350`. If not `width = 300`.

[__src/App.js__](https://github.com/pksjce/jest-blog-samples/blob/master/window-testing/src/App.js)

```javascript

import React, {Component} from 'react';

export const WIDTH_ABOVE_1024 = 350;
export const WIDTH_BELOW_1024 = 300;
export const THRESHOLD_WIDTH = 1024;

export default class App extends Component {
    constructor() {
        super();
        this.state = {
            width: window.innerWidth > THRESHOLD_WIDTH ? WIDTH_ABOVE_1024: WIDTH_BELOW_1024
        }
        this.handleResize = this.handleResize.bind(this);
    }
    componentDidMount() {
        window.addEventListener('resize', this.handleResize)
    }
    componentWillUnmount() {
        window.removeEventListener('resize', this.handleResize)
    }
    render() {
        return <div style={{width:this.state.width}}>My width is {this.state.width}</div>;
    }
    handleResize() {
        this.setState({
            width: window.innerWidth > THRESHOLD_WIDTH ? WIDTH_ABOVE_1024 : WIDTH_BELOW_1024
        })
    }
}
```

For the above component, we again need to test if

 - Default behaviour.
 - Resize to `width > 1024`
 - Resize to `width <=1024`

 [__src/App.test.js__](https://github.com/pksjce/jest-blog-samples/blob/master/window-testing/src/App.test.js)

```javascript
import React from 'react';
import App, {WIDTH_ABOVE_1024, WIDTH_BELOW_1024, THRESHOLD_WIDTH} from './App';
import {mount} from 'enzyme';

describe('The main app', () => {
    let app;
    beforeEach(() => {
        app = mount(<App/>, {attachTo: document.getElementById('root')});
    })

    it('the app should have text', () => {
        var width = window.innerWidth > THRESHOLD_WIDTH ? WIDTH_ABOVE_1024 : WIDTH_BELOW_1024;
        expect(app.text()).toBe("My width is " + width);
        app.unmount();
    })
    it ('should change text on click', () => {
        window.resizeTo(1000, 1000);
        expect(app.text()).toBe("My width is " + WIDTH_BELOW_1024);
        window.resizeTo(1025, 1000)
        expect(app.text()).toBe("My width is " + WIDTH_ABOVE_1024);
        app.unmount();
    })
})
```

 Things to note are

  - We can totally use the component's constants for testing. 
  - Once you `mount` your app using `enzyme`, you will also need to `unmount()` it at the end of each test.
  - `window.resizeTo` will trigger a change in `window.innerWidth`. The app code rerenders as per app logic and we can test for printed width.

### Key Takeaways

- Testing is now way more easier than before with `Jest`
- DOM testing with `react` needs `enzyme` as a renderer util and `jsdom` as a DOM helper.
- Mocking can be tricky. It's fine to have hacky mocks.

I hope that this article has inspired you to take up testing your application with Jest and you will be motivated to add it to your dev workflow!
