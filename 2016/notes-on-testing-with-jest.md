# Notes on Jest

During the past 4 months, I have been working on the Flipkart desktop website team. For me it meant finally working on code at scale. If there was any place where TDD would be appreciated, it was here, on user facing critical code. The tests were being written on a setup of Karma, Mocha, Sinon and Enzyme. It worked pretty great, but the major peeve for all of us, was that, it was so slow!

The team had already tried using Jest before, had faced many issues with the automocking and given up. But then Jest made a turnaround and with a great community effort released improved and [new versions](https://facebook.github.io/jest/blog/2016/10/03/jest-16.html). From all the blogposts, we found that Jest's main focus was testing performance and something new called ["Snapshot testing"](https://facebook.github.io/jest/blog/2016/07/27/jest-14.html). Also, mainly it promised to be "Painless JavaScript Testing". Jest can mostly work for testing in any JavaScript framework or environment. Also, its a known fact that Jest uses itself to test it's codebase.

This post wants to be an FAQ guide which would have given us extra help during this transition.

## FAQs

The FAQs are categorised by the facet of testing that they cover. These may also apply to any testing in general.

### Fundamentally speaking

__Why should we move to `Jest` when we already have a working setup?__

A move to Jest would mean a somewhat change in paradigm and would involve team effort. We justified the effort with the following reasons

  - Jest is way faster than our current setup. It runs test suites in a parallel and the community is focused on improving developer experience through performance.
  - I really liked the interactive CLI that comes with it. In `watch` mode, Jest runs only the tests associated with your code changes. But it allows you to interact with the tests by providing functionality to update snapshots, search for a particular suite, run all tests if you want to.
  - [Great documentation](https://facebook.github.io/jest/docs/getting-started.html)
  - Snapshot testing for components. It is kind of tailor made for React apps.
  - Great defaults meaning "Zero" configuration. Running `jest` alone would be enough to run test files. Mocking is not a marriage of complexity and confusion.
  - Existing tests could be easily migrated [manually or through automated codemods](https://facebook.github.io/jest/docs/migration-guide.html#content)

__What about testing on the browser?__

Yes, the earlier setup opened a chromium browser and tested our components. It was pretty cool actually. But, it was one of the things that made executing tests slow for us. Being priviledged enough to have manual QA, we decided to forego this functionality.

For more reasoning, you should [read this](https://github.com/facebook/jest/issues/139#issuecomment-229277654) by [@cpojer](https://github.com/cpojer) on why Facebook doesn't need them.
Also, we are open to using [React storybook](https://getstorybook.io/) in the future, which I think has potential for browser testing.

__What is Snapshot testing? Will it really help?__

Snapshot testing, is a procedure in which, Jest generates readable snapshots of your component's view. A snapshot is basically a text file with xml, which acts like the source of your view, at that point in your testcase. Different snapshots can be created with different props to your components.

Snapshots can be saved after simulating events like `click` and `hover`. The point would be to save the expected view of your component which is a function of some mock data. If in the future, a developer changes your component, Jest would create a new snapshot of the component and compare it with the saved snapshot. If there is a change in view, it will alert the developer and she will have to tell Jest to save this new snapshot. Since the snapshot is committed to version control, it will show up in a code review and the changes can be approved accordingly.
While we still have reservations of, if its actually feasible to review large changes in the snapshot, I definitely think this enriches the content of a code review.(**Edit** [@cpojer](https://github.com/cpojer) said in this [comment](https://github.com/kosamari/web.advent.today/pull/27#discussion_r91407888) that snapshots are even more powerful!)

### Implementation Questions

__How do I test my React+Redux code?__

We have been following an approach where we test each of the moving parts independently.

Components
----------
A React component is a view only component, with some event handling and some local state. It is `connect`ed to the Redux store for state and actions.
We test the base component, which is free of Redux. For a component like,

__my-component.js__

```javascript
import React, {Component} from 'react';
import {connect} from 'react-redux';

export class MyComponent extends Component {
    constructor () {
    super();
    this.state = {
        localProp: 'Default'
    };
    }
    handleClick = () => {
    this.setState({
        localProp: 'After click'
    });
    }
    render () {
    return (<div>
        <div>{this.props.reduxProp}</div>
        <div>{this.state.localProp}</div>
        <div onClick={this.handleClick}>Click me</div>
    </div>);
    }
}

export default connect(reduxState => ({reduxProp: reduxState.prop}) )(MyComponent)
```

__my-component.test.js__

```javascript
// needed for jsx
import React from 'react';
// import the named export
import {MyComponent} from './my-component';
// enzyme has rendering utils
import {shallow} from 'enzyme';
// needed to create snapshot
import {shallowToJson} from 'enzyme-to-json';

describe('<MyComponent />', () => {
    let component;
    // create an instance of MyComponent before each test
    beforeEach(() => {
        component = shallow(<MyComponent reduxProp="A test prop" />);
    });
    // test rendering expectations
    it('should render well', () => {
        // DOM test
        expect(component.contains(<div>A test prop</div>)).toBe(true);
        expect(component.contains(<div>Default</div>)).toBe(true);
        // Match snapshots (creates the first time)
        expect(shallowToJson(component)).toMatchSnapshot();
    });
    // test the action
    it('should change text after click', () => {
        //find particular DOM and simulate click
        component.childAt(2).simulate('click');
        test expectation after click is done
        expect(component.text().indexOf('After click') > -1).toBe(true);
        // match snapshot because view changed.
        expect(shallowToJson(component)).toMatchSnapshot();
    });
});
```

Reducers
--------

Reducers, being pure functions, are ridiculously easy to test. The redux docs [shows how to do this](http://redux.js.org/docs/recipes/WritingTests.html#reducers).

Actions
-------

We used the [recommended technique](http://redux.js.org/docs/recipes/WritingTests.html#async-action-creators), but instead of using `nock` we decided to mock the `fetch` api with Jest's easy mocking system.

__asyncUtil.js__

```javascript
import configureMockStore from 'redux-mock-store';
import thunk from 'redux-thunk';
import fetchMock from 'fetch-mock';

export const runsynctest = ({testName, expectedAction, fn, params}) => {
    it(testName, () => {
        expect(fn.call(null, ...params)).toEqual(expectedAction);
    });
};
export const runasynctest = ({testName, expectedActions, fn, params}) => {
    const middlewares = [ thunk ];
    const mockStore = configureMockStore(middlewares);
    const store = mockStore();
    fetchMock.mock('*', params); // This can be customised
    it(testName, () => {
    return store
        .dispatch(fn.call(null, ...params))
        .then(args => {
            expect(store.getActions()).toEqual(expectedActions);
        });
    });
};
```

__my-action.test.js__

```javascript
import { runasynctest, runsynctest } from './actionsUtil';
// fetchVal is your typical fetch action which returns a promise
import { fetchVal } from '../my-actions';
import * as types from 'constants/my-constants';

describe('async-actions', () => {
    runasynctest(
    {
        testName: 'should simulate fetch action',
        expectedActions: [
        { type: types.GET__REQUEST },
        {
            type: types.GET__SUCCESS,
            pids: [1, 2]
        }
        ],
        fn: fetchVal,
        params: [[1, 2]]
    });
});
```

Components with Actions
-----------------------

If in the above example for `MyComponent`'s testing, the `handleClick` were to send actions with redux like

__my-component.js__

```javascript
...

handleClick = event => {
    this.props.fetchValAction(event);
}

...
```

__my-component.test.js__

```javascript
describe('<MyComponent />', () => {
    it('should call fetchVal action', () => {
    const mockFetchVal = jest.mock();
    const component = mount(<MyComponent fetchValAction={mockFetchVal}/>);
    component.childAt(2).simulate('click');
    expect(mockFetchVal).toHaveBeenCalled();
    })
})
```

tldr; Various solutions can be adopted. Choose whichever suits your environment.

__How do I test asynchronous functions?__

The Jest docs has a [tutorial on asynchronous testing](https://facebook.github.io/jest/docs/tutorial-async.html#content). Converting them to `Promises` works too.

__How to simulate window events?__

We used [jsdom](https://github.com/tmpvar/jsdom) to get a DOM that works with Node.js.
We set it up for the project at Jest's [`setupFile`](https://facebook.github.io/jest/docs/configuration.html#setupfiles-array). 

__setupFile.js__

```javascript
const jsdom = require('jsdom');

const documentHTML = '<!doctype html><html><body><div id="root"></div></body></html>';
global.document = jsdom.jsdom(documentHTML);
global.window = document.parentWindow;
global.window.resizeTo = (width, height) => {
	global.window.innerWidth = width || global.window.innerWidth;
	global.window.innerHeight = width || global.window.innerHeight;
	global.window.dispatchEvent(new Event('resize'));
};
global.Promise = require.requireActual('promise');

``` 

Now we can simulate resize by the mock method above. So we have tests like below.

__example__

```javascript
/*
  The test below checks if snapshots were the same before and after resizes.
  You can also test for other window events like scroll in this manner.
*/
it('should resize properly', () => {
  const props = { highlights: true };
  const component = mount(<MyComponent {...props} />, {attachTo: document.getElementById('root')});
  global.window.resizeTo(1025);
  expect(mountToJson(component)).toMatchSnapshot();
  global.window.resizeTo(1020);
  expect(mountToJson(component)).toMatchSnapshot();
});
```

__I can't get PostCSS to work!__

This was one area where Jest fell short of expectations. `karma` allows you directly put in `webpack` transformations onto the configuration. `Jest` has [equivalent configuration](https://facebook.github.io/jest/docs/tutorial-webpack.html#content) which is mapped to `webpack` config. But we found it to be adequate for our needs except that we could not get PostCSS plugins to work. Babel preprocessing is done out of the box using [`babel-jest`](http://facebook.github.io/jest/docs/getting-started.html#babel-integration) which comes bundled with Jest in the latest version. Jest can support [some preprocessors](https://facebook.github.io/jest/docs/configuration.html#transform-object-string-string), but somehow cannot support PostCSS as it has asynchronous plugins.
So we had to use this proxy called [identity-obj-proxy](https://github.com/keyanzhang/identity-obj-proxy) to proxy our styles with the given className.

An alternative would be to try [this advice](https://github.com/kosamari/web.advent.today/pull/27#discussion_r91408412) to preprocess all the css for easy consumption.

For example for JSX,

`<div className={styles['button-class']}>Click</div>`

the below snapshot is generated,

`<div className='button-class'>Click</div>`

It does not allow you to test for true styles, only classNames. But, its a tradeoff we have made.

__Can I calculate coverage?__

I'm glad you asked the question! Coverage stats are inbuilt within Jest. No configuration is required. `jest --coverage` does it.
The downside is that it considerably slows down your test runs. Hence we don't keep it on during development. There are also really cool features like coverage thresholds. You can learn this by watching this excellent video on [Egghead.io by @kentcdodds](https://egghead.io/lessons/javascript-track-project-code-coverage-with-jest).

## Conclusion

Its been a couple of weeks since we started using `Jest`. Code Coverage is slowly growing within the team. However we have configured our PR builds to fail if existing tests fail, that includes snapshots. In the future, we plan to have coverage thresholds too!