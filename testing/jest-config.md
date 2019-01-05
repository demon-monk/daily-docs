# jest-config

## setup a basic jest test

- 准备好被测的工程，创建源文件
- 写好测试用例，放在`__tests__`目录下
- 安装`jest`

```sh 
yarn add jest -D
```

- 支持babel转义

```sh
yarn add @babel/core @babel/preset-env babel-core@^7.0.0-bridge.0  babel-jest regenerator-runtime
```

```js
// .babelrc.js
const isTest = String(process.env.NODE_ENV) === 'test'
module.exports = {
  presets: [
      [
          '@babel/preset-env', {
      		modules: isTest ? 'commonjs' : false // 1
  		   }
      ], 
      '@babel/preset-react'
  ],
}
```

1. test环境下打开babel的模块转义，转义为commonjs模块。其他环境关闭，为了使用webpack的`tree-shaking`特性

## jest.config.js

### environment

jsdom实现了浏览器环境对象`Window`

```js
module.exports = {
    testEnvironment: 'jest-environment-jsdom' // default
    // 'jest-environment-node
}
```

### moduleNameMapper

#### map resource to self-defined js module

在webpack构建工程时，我们可以把任何资源都看作一个模块，通过各种各样的loader去加载。但是在jest运行的时候，并没有这些loader，所以在js中引入像`.css`这样的资源可能会报错。

配置文件中`moduleNameWrapper`属性就是为了解决这个问题。虽然没有loader，但是我们可以将某种类型的资源，比如`css`，替换为一个js module，这样就可以加载了。

```js
module.exports = {
    moduleNameMapper: {
        '\\.css$': require.resolve('./test/style-mock.js')
    }
}
```

然后再`'./test/style-mock.js'`中写入

```js
export default {}
```

即可。

#### map resource to `identity-obj-proxy`

当你使用css module时，使用👆将`.css`映射成一个空的模块，会导致你render出来的snapshot缺少一些className，（`styles.autoScalingText`是`undefined`）

```jsx
import styles from './auto-scaling-text.module.css'

class AutoScalingText extends React.Component {
    render() {
    const scale = this.getScale()

    return (
      <div
        className={styles.autoScalingText}
        style={{transform: `scale(${scale},${scale})`}}
        ref={this.node}
      >
        {this.props.children}
      </div>
    )
  }
}
```

解决这个问题的办法是，当使用css module(`*.module.css`)时，使用`identity-obj-proxy`将要映射的值设为属性名。

```js
module.exports = {
    moduleNameMapper: {
        '\\.module\\.css$': 'identity-obj-proxy',
        '\\.css$': require.resolve('./test/style-mock.js'),
    }
}
```

这时候再看渲染出的snapshot，可以发现className为`autoScalingText`。虽然跟真实情况不一样（没有hash），但是从中可以看出一些异动。

### snapshot 

本质：就是讲一个对象序列化，方便在更改的时候看出不同。

在使用`react-testing-library`的对组件进行渲染时，时可以拿到render出来的container的`innerHTML`的。通过`innerHTML`也可以看出每次改动对结果的影响。但是不是很直观，只要有一点改动，整个HTML都会变化，所以需要序列化。

测试用例如下

```js
import 'react-testing-library/cleanup-after-each'
import React from 'react'
import {render} from 'react-testing-library'
import AutoScalingText from '../auto-scaling-text'

test('renders', () => {
    const {container} = render(<AutoScalingText />)
    expect(container).toMatchSnapshot()
})
```

每次改变确认无误后，只需`yarn test -u`即可更新snapshot

#### remove wrapper div

React组件返回的结构通常只有一个`div`，而`react-testing-library.render（）`返回的container会再加一层`div`。可以使用

```js
expect(container.firstChild).toMatchSnapshot()
```

来减少一层snapshot的嵌套。

#### add customize serializer for emotion

Emotion是一个`CSS-in-JS`库，通过它我们可以直接在js中写CSS。通过将emotion转化后，div标签上的`css=...`其实是`class="css-hashtag"`，只要css的值有变化，这个hash tag就会变化。

```js
class CalculatorDisplay extends React.Component {
 	render() {
        return (
          <div
            {...props}
            css={{
              color: 'white',
              background: '#1c191c',
              lineHeight: '130px',
              fontSize: '6em',
              flex: '1',
            }}
          >
            <AutoScalingText>{formattedValue}</AutoScalingText>
          </div>
        )
      }   
}
```

这样就会给测试带来麻烦，因为只看hash值是看不出来到底改动了那些东西的。

通过使用`jest-emotion`作为snapshotSerializer，就可以将`css=…`转化为`class="emotion0"`和css的值。这样以后，每次改动，不变的是`class`，变化的是`.emotion0`对应的值。snapshot更有意义。

```html
.emotion-0 {
  color: white;
  background: #1c191c;
  line-height: 130px;
  font-size: 6em;
  -webkit-flex: 1;
  -ms-flex: 1;
  flex: 1;
}

<div
  class="emotion-0"
>
</div>
```



#### snapshot for lazy-loaded module

有时候我们会使用动态加载

```js
// src
import loadable from 'react-loadable'
const CalculatorDisplay = loadable({
  loader: () => import('calculator-display').then(mod => mod.default),
  loading: () => <div style={{height: 120}}>Loading display...</div>,
})
export class Calculator extends React.Component {
  // render <CalculatorDisplay /> in exported component  
}
```

我们对`Calculator`这个组件进行snapshot

```js
import React from 'react'
import {render} from 'react-testing-library'
import Calculator from '../calculator'

test('renders calculator', () => {
    const {container, debug} = render(<Calculator/>)
    debug(container)
    expect(container.firstChild).toMatchSnapshot()
})
```

可以看到`<CalculatorDisplay/>`对应的那部分snapshot总是

```html
<div
     style="height: 120px;"
>
    Loading display...
</div>
```

因为在测试的时候动态加载还没完成。

为了能测到加载完成的状态，我们可以等待`loadable`完成

```js
import loadable from 'react-loadable'
test('renders calculator', async () => {
    await loadable.preloadAll()
    // 同上
})
```

#### jest module resolution

按照👆的写法，测试会报错。在webpack中，我们把`src/shared`目录写入了`resolve`中，随意可以直接写下面的模块名`calculator-display`。但是jest中会把他当成一个node module，但是它不是，所以会报错。

同webpack配置中的`resolve`一样，jest也有一个类似的配置属性`moduleDirectories`，与前者填一样的值即可

```js
// jest.config.js
module.exports = {
	// same as above
    moduleDirectories: ['node_modules', path.join(__dirname, 'src'), 'shared'],

}
```

#### jest dynamic import

jest经过👆配置后，能够正确访问到`calculator-display`，但是还是会报错`Not support`。因为在`Jest/Node/CommonJS`环境中是不支持动态import的。babel配置中，在Test环境下加上`babel-plugin-dynamic-import-node`插件就可以了。

### extract common part 

`import 'react-testing-library'`， 给expect增加snapshot serializer，这些操作可能是很多测试用例文件都需要用到的。最好能有一个统一的方式，在所有的测试用例开始执行前进行操作。

我们创建一个`test/setup-tests.js`来存放这些操作。

```js
import 'react-testing-library/cleanup-after-each'
import {createSerializer} from 'jest-emotion'
import * as emotion from 'emotion'

expect.addSnapshotSerializer(createSerializer(emotion))
```

`jest.config.js`中提供了`setupTestFrameworkScriptFile`字段，用于存放jest加载完毕后要执行的脚本。

```js
// jest.config.js
module.exports = {
    // same as above...
    // before jest is loaded
    // testFiles: [],
    // after jest is loaded
    setupTestFrameworkScriptFile: require.resolve('./test/setup-tests.js'),

}
```

然后再所有的测试用例中删除这些操作。

### test components driving by provider

有些组件需要通过`Provider`提供props，这时我们在测试的时候就需要带上`Provider`

```js
import {ThemeProvider} from 'emotion-theming'
import {dark} from '../../themes'

test('renders calculator',  () => {
    const {container} = render(
        <ThemeProvider theme={dark}>
            <Calculator value="0" />
        </ThemeProvider>
    )
    expect(container.firstChild).toMatchSnapshot()
})
```

#### optimization

可以将引入Provider的操作封装到一个render方法中，然后就可以使用这个`render`方法代替原来`react-testing-library`的`render`方法。

创建`test/calculator-test-utils`

```js
import React from 'react'
import {render} from 'react-testing-library'
import {ThemeProvider} from 'emotion-theming'
import {dark} from '../src/themes'

export function renderWithProviders(ui, options) {
    return render(
        <ThemeProvider theme={dark}>{ui}</ThemeProvider>
    )
}
```

在测试用例中使用`renderWithProviders`代替原来的`render`

```js
import { renderWithProviders } from '../../../test/calculator-test-utils';

test('renders calculator',  () => {
    const {container} = renderWithProviders(<Calculator value="0"/>)
    expect(container.firstChild).toMatchSnapshot()
})
```

#### futher optimization
将`react-testing-library`的其他部分移植到`test/calculator-test-utils`中
```js
// test/calculator-test-utils
// the same as above
export * from 'react-testing-library'
export {renderWithProviders as render}
```
将`test`目录下的模块置于jest可以直接访问的位置，一遍在测试用例中使用`node_module`的方式引入`test/calculator-test-utils`
```js
//jest.config.js
module.exports = {
    moduleDirectories: 
        [
            // the same as above
            path.join(__dirname, 'test'),
        ],
}
```

```js
//__tests__
import { render } from 'calculator-test-utils';

test('renders calculator',  () => {
    const {container} = render(<Calculator value="0"/>)
    expect(container.firstChild).toMatchSnapshot()
})
```
👆`calculator-test-utils`下面会有横线，提示找不到路径。我们需要将`jest.config.js`中配置的模块路径重写到`eslintrc`中。
```sh
yarn add eslint-import-resolver-jest -D
```

```js
// .eslintrc.js
const path = require('path')
module.exports = {
  extends: [...],
  overrides: [
    {
      files: ['**/__tests__/**'],
      settings: {
        'import/resolver': {
          jest: {
            jestConfigFile: path.join(__dirname, 'jest.config.js')
          }
        }
      }
    }
  ]
}

```