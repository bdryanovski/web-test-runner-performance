# Web Test Runner Performance

The Web Test Runner Performance provides plugins for [web-test-runner](https://modern-web.dev/docs/test-runner/overview/) to measure UI performance as well as a custom reporter for logging results. The package provides three plugins for your `web-test-runner` configuration.

- `bundlePerformancePlugin`: measure bundle/package size output
- `renderPerformancePlugin`: measure component render performance
- `performanceReporter`: for writting performance results to disk

## Setup

Below you can find a minimal setup. To use `web-test-runner-performance` create a standalone test runner separate from your standard test runner config. Performance tests should be run independent of other tests and only run one test and browser at a time for the most accurate results.

```javascript
// web-test-runner.performance.mjs
import { defaultReporter } from '@web/test-runner';
import { esbuildPlugin } from '@web/dev-server-esbuild';
import { playwrightLauncher } from '@web/test-runner-playwright';

export default ({
  concurrency: 1,
  concurrentBrowsers: 1,
  nodeResolve: true,
  testsFinishTimeout: 20000,
  files: ['./src/*.performance.ts'],
  browsers: [playwrightLauncher({ product: 'chromium', launchOptions: { headless: false } })],
  reporters: [
    defaultReporter({ reportTestResults: true, reportTestProgress: true })
  ],
  plugins: [
    esbuildPlugin({ ts: true, json: true, target: 'auto', sourceMap: true }),
  ],
});
```
## Bundle Performance

The `testBundleSize` takes a JavaScript module and will bunle any imports including CSS providing
a final output size in Kb. The bundle is generated by Rollup and will treeshake and minify the code.
This is helpful for testing library code ensuring it properly treeeshakes.
You can also pass an alias config to test local file paths.

```javascript
// web-test-runner.performance.mjs
...
import { bundlePerformancePlugin } from 'web-test-runner-performance';

export default ({
  ...
  plugins: [
    bundlePerformancePlugin(),
  ]
});
```

```javascript
// component.performance.js
import { expect } from '@esm-bundle/chai';
import { testBundleSize } from 'web-test-runner-performance/browser.js';

describe('performance', () => {
  it('should meet maximum css bundle size limits (0.2kb brotli)', async () => {
    expect((await testBundleSize('./demo-module/index.css')).kb).to.below(0.2);
  });

  it('should meet maximum js bundle size limits (0.78kb brotli)', async () => {
    expect((await testBundleSize('./demo-module/index.js')).kb).to.below(0.8);
  });
});
```

You can also pass in an alias configuration to alias imports for local packages.

```javascript
bundlePerformancePlugin({
  aliases:  [{ find: /^demo-module$/, replacement: `${process.cwd()}/demo-module` }]
})
```

## Render Performance

The `renderPerformancePlugin` will measure the render time of a given custom element in milliseconds.

```javascript
// web-test-runner.performance.mjs
...
import { renderPerformancePlugin } from 'web-test-runner-performance';

export default ({
  ...
  plugins: [
    esbuildPlugin({ ts: true, json: true, target: 'auto', sourceMap: true }),
    renderPerformancePlugin(),
  ],
});
```

Once the plugin is added a test can be written to check how quick an element can be rendered.

```javascript
// component.performance.js
import { expect } from '@esm-bundle/chai';
import { testBundleSize, testRenderTime, html } from 'web-test-runner-performance/browser.js';

describe('performance', () => {
  it(`should meet maximum render time 1000 <p> below 50ms`, async () => {
    const result = await testRenderTime(html`<my-element>hello world</my-element>`, { iterations: 1000, average: 10 });
    expect(result.averages.length).to.eql(10);
    expect(result.duration).to.below(50);
  });
});
```

The `testRenderTime` takes a template of the component to render and will return how long the 
element took to render its first paint in miliseconds. The config accepts an `iterations` property to render
the template n+ iterations. By default the test runner will render three times and
average the results. This can be customized using the `average` property.

## Reporter

The `performanceReporter` provides a custom reporter which will write all test
results to a json file.

```javascript
// web-test-runner.performance.mjs
import { playwrightLauncher } from '@web/test-runner-playwright';
import { esbuildPlugin } from '@web/dev-server-esbuild';
import { defaultReporter } from '@web/test-runner';
import { bundlePerformancePlugin, renderPerformancePlugin, performanceReporter } from 'web-test-runner-performance';

export default ({
  concurrency: 1,
  concurrentBrowsers: 1,
  nodeResolve: true,
  testsFinishTimeout: 20000,
  files: ['./src/*.performance.ts'],
  browsers: [playwrightLauncher({ product: 'chromium', launchOptions: { headless: false } })],
    reporters: [
    defaultReporter({ reportTestResults: true, reportTestProgress: true }),
    performanceReporter({ writePath: `${process.cwd()}/dist/performance` })
  ],
  plugins: [
    esbuildPlugin({ ts: true, json: true, target: 'auto', sourceMap: true }),
    renderPerformancePlugin(),
    bundlePerformancePlugin(),
  ]
});
```

```json
// dist/performance/report.json
{
  "renderTimes": [
    {
      "testFile": "/src/index.performance.ts",
      "duration": 27.559999999999995
    }
  ],
  "bundleSizes": [
    {
      "testFile": "/src/index.performance.ts",
      "compression": "brotli",
      "kb": 0.193
    },
    {
      "testFile": "/src/index.performance.ts",
      "compression": "brotli",
      "kb": 0.78
    }
  ]
}
```

Learn more about getting started here https://coryrylan.com/blog/testing-web-performance-with-web-test-runner