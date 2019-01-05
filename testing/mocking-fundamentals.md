# Jest Mocking

## monkey-patching

[monkey-patching](https://stackoverflow.com/questions/5626193/what-is-monkey-patching), 临时（动态）将某个变量的值替换为另外一个值，测试完后再还原为原来的值。

```js
 const thumbWar = require('../thumb-war')
+const utils = require('../utils')
+
+const originGetWinner = utils.getWinner
+utils.getWinner = p1 => p1
+
 test('bang should win', () => {
   const winner = thumbWar('Bang', 'Dan')
   expect(winner).toEqual('Bang')
+  utils.getWinner = originGetWinner
 })
```

## jest.fn

使用指定的方式进行`monkey-patching`，以方便记录被Mock的函数的执行过程信息（调用了几次，每次调用的参数等）。

```js
 const utils = require('../utils')
 
 const originGetWinner = utils.getWinner
-utils.getWinner = p1 => p1
+utils.getWinner = jest.fn(p1 => p1)
 
 test('bang should win', () => {
   const winner = thumbWar('Bang', 'Dan')
   expect(winner).toEqual('Bang')
+  expect(utils.getWinner).toHaveBeenCalledTimes(2)
+  expect(utils.getWinner).toHaveBeenNthCalledWith(1, 'Bang', 'Dan')
+  expect(utils.getWinner).toHaveBeenNthCalledWith(2, 'Bang', 'Dan')
+  console.log(utils.getWinner)
   utils.getWinner = originGetWinner
 })

```

### jest.fn implementation

根据上述对`jest.fn`功能的描述可以发现，只需要在mock实现的外面包一层记录每次调用信息的逻辑(1)就好，并使用一个属性把这些信息存储起来

```js
function fn(impl) {
  const mockFn = (...args) => {
    mockFn.mock.calls.push(args) // 1
    return impl(...args)
  }
  mockFn.mock = { calls: [] } // 2
  return mockFn
}

module.exports = { fn }
```

## jest.spyOn

每次测试都要记得先保存原始实现，测试结束后再讲原始实现赋值回去。这样做很烦，`jest.spyOn`解决了这个麻烦

1. `jest.spyOn`将模块的某个方法替换为一个空实现
2. 这个空实现有一个方法`mockImplementation`可以用来指定mock内容
3. 还有另外一个方法`mockRestore`将mock的对象还原

```js
const utils = require('../utils')
const thumbWar = require('../thumb-war')

test('return winner', () => {
  jest.spyOn(utils, 'getWinner') // 1
  utils.getWinner.mockImplementation(p1 => p1) // 2
  const winner = thumbWar('Bang', 'Dan')
  expect(winner).toEqual('Bang')
  utils.getWinner.mockRestore() // 3
})

```

### jest.spyOn implementation

1. 增加了`fn`返回值的`mockImplementation`方法，在`fn`中实现
2. 在`spyOn`中将`fn`返回的值增加了`mockRestore`方法

```js
function fn(impl = () => { }) {
  const mockFn = (...args) => {
    mockFn.mock.calls.push(args)
    return impl(...args)
  }
  mockFn.mock = { calls: [] }
  mockFn.mockImplementation = newImpl => (impl = newImpl) // 1
  return mockFn
}

function spyOn(obj, prop, impl = () => {}) {
  const originVal = obj[prop]
  obj[prop] = fn(impl)
  obj[prop].mockRestore = () => (obj[prop] = originVal) // 2
}
module.exports = { fn, spyOn }
```

## mock js module

上面例子中，我们使用`fn`和`spyOn`的方式，其实还是`monkey-patching`，不过是在引入目标模块后，对其进行修改，并在测试结束后进行还原。

这种方法仅在`commonjs`中可用，在`esModule`中无法使用。所以jest提供了专门的对模块进行Mock的方法。

### inline: jest.mock

```js
const thumbWar = require('../thumb-war')
const utilsMock = require('../utils')

jest.mock('../utils.js', () => {
  return {
    getWinner: jest.fn(p1 => p1),
  }
})

test('returns winner', () => {
  const winner = thumbWar('Bang', 'Dan')
  expect(winner).toEqual('Bang')
  expect(utilsMock.getWinner.mock.calls).toEqual([
    ['Bang', 'Dan'],
    ['Bang', 'Dan'],
  ])
  utilsMock.getWinner.mockReset()
})

```

### jest.mock implementation with require.cache

1. 首先引入👆已经实现过的`fn`
2. 使用`require.cache`将要mock的模块对象的`.exports`属性替换掉
3. 然后再引入这个模块时，拿到的就是被Mock完了的模块了
4. 测试完后记得把`require.cache`删除，以免影响其他地方使用这个模块

❓❓❓

​		我们的实现里，必须先替换`require.cache`，然后再`require`要mock的模块，才能保证引入的模块是已经被替换过的。为啥jest可以先`require`，再`.mock`

💡💡💡

​		jest再run测试用例之前会先做一遍代码转化，把`jest.mock`这种语句提前

```js
// use node src/__tests__/inline-module-mock-impl.js to test this file instead of yarn test
// because jest will control the module system and require.cache operation will not effect
// 1
const { fn } = require('../__testUtils/utils')
const utilsPath = require.resolve('../utils')
// 2
require.cache[utilsPath] = {
  id: utilsPath,
  filename: utilsPath,
  loaded: true,
  exports: {
    getWinner: fn(p1 => p1),
  },
}
const assert = require('assert')
const thumbWar = require('../thumb-war')
// 3
const utils = require('../utils')

const winner = thumbWar('Kent C. Dodds', 'Ken Wheeler')
assert.strictEqual(winner, 'Kent C. Dodds')
assert.deepStrictEqual(utils.getWinner.mock.calls, [
  ['Kent C. Dodds', 'Ken Wheeler'],
  ['Kent C. Dodds', 'Ken Wheeler'],
])

// 4
// cleanup
delete require.cache[utilsPath]

```

### external mock module

使用inline的方式对模块进行mock有一个确定就是，对同一模块再不同的测试文件中使用到，都必须要做一次mock实现。

为了避免这种情况，jest还支持外部模块mock。

1. 在被mock的模块同级创建一个`__mocks__`目录，并在内部创建一个跟被mock模块同名的js文件

   ```
   src
   ├── __mocks__
   │   └── utils.js
   ├── __tests__
   │   ├── external-module-mock.js
   ├── thumb-war.js
   └── utils.js
   ```

   ```js
   // src/__mocks__/utils.js
   module.exports = {
     getWinner: jest.fn(p1 => p1),
   }
   ```

2. 在测试用例中正常引入模块测试，`jest.mock`调用时不需要给出mock实现

3. 用完`.mockReset()`

   ```js
   const thumbWar = require('../thumb-war')
   const utils = require('../utils')
   
   jest.mock('../utils.js') // 2 no need to provide mock implementation
   
   test('returns winner', () => {
     const winner = thumbWar('Bang', 'Dan')
     expect(winner).toEqual('Bang')
     expect(utils.getWinner.mock.calls).toEqual([['Bang', 'Dan'], ['Bang', 'Dan']])
     utils.getWinner.mockReset() // 3
   })
   
   ```

### external mock module implementation

1. 这里`__mocksWithoutFramework__`相当于`__mocks__`
2. 先引入下mock内容，让这个路径下的`require.cache`有值
3. 将真实utils的`require.cache`替换为mock模块的`require.cache`
4. 测试完删除`require.cache`

```js
require('../__mocksWithoutFramework__/utils') // 1 2
const utilsPath = require.resolve('../utils.js')
const utilsMockPath = require.resolve('../__mocksWithoutFramework__/utils.js')
require.cache[utilsPath] = require.cache[utilsMockPath] // 3

const assert = require('assert')
const thumbWar = require('../thumb-war')
const utils = require('../utils')

const winner = thumbWar('Bang', 'Dan')
assert.strictEqual(winner, 'Bang')
assert.deepStrictEqual(utils.getWinner.mock.calls, [
  ['Bang', 'Dan'],
  ['Bang', 'Dan'],
])
delete require.cache[utilsPath] // 4
```

