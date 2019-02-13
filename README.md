# How Webpack Load Modules And Chunks

1. `index.html` loads three js files.

    ```html
    <script type="text/javascript" src="./runtime.js"></script>
    <script type="text/javascript" src="./vendor.js"></script>
    <script type="text/javascript" src="./app.js"></script>
    ```

1. `app.js` and `vendor.js` loaded

    Now we have an array which store information of module and chunk:

    ```js
    window['myJsonp'] = [
      // [[...chunkIds], {...moreModules}, [executeModule, ...deferredModules]]
      [['app'], {module_a, module_b, module_c, entry_point}, [['entry_point', 'vendor']]],
      [['vendor'], {vendor_a, vendor_b, vendor_c}],
    ];
    ```
1. `runtime.js` loaded

    The following code execute:

    ```js
    var jsonpArray = window["myJsonp"] = window["myJsonp"] || [];
    var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
    jsonpArray.push = webpackJsonpCallback;
    jsonpArray = jsonpArray.slice();
    for(var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
    var parentJsonpFunction = oldJsonpFunction;
    ```

    * Assign `window['myJsonp']` to `jsonpArray` variable.
    * Assign `window['myJsonp'].push` to `oldJsonpFunction` also to `parentJsonpFunction`.
    * Replace `window['myJsonp'].push` with `webpackJsonpCallback`.
    
    Call `webpackJsonpCallback` with each elements of `jsonpArray`. [[goto](https://github.com/tpai/how-webpack-load-modules-and-chunks/blob/master/runtime.js#L3)]

    ```js
    webpackJsonpCallback([['app'], {...}, [['entry_point', 'vendor']]])
    webpackJsonpCallback([['vendor'], {...}])
    ```

    Install chunks and assign modules.

    ```js
    installedChunks = {'runtime': 0, 'app': 0, 'vendor': 0};
    modules = {module_a, module_b, module_c, entry_point, vendor_a, vendor_b, vendor_c};
    ```

    Push `executeModules`(entry_point, vendor) into `deferredModules`.

    ```js
    deferredModules = [['entry_point', 'vendor']];
    ```

    To fulfill the condition of entry point execution, `checkDeferredModules` will check the required chunks [[goto](https://github.com/tpai/how-webpack-load-modules-and-chunks/blob/master/runtime.js#L37)]

1. Execute entry point

    Chunk `vendor` has been installed, so execute `entry_point` module.

    ```js
    if(fulfilled) {
      result = __webpack_require__(__webpack_require__.s = deferredModule[0]);
    }
    ```

    All module code are wrapped to text and execute by `eval`.

    ```js
    entry_point: (function(module, __webpack_exports__, __webpack_require__) {
      eval(`
        var vendor_c__IMPORTED_MODULE__ = __webpack_require__('vendor_c');
        vendor_c__IMPORTED_MODULE__('entry point loaded');
    ```

    Before compilation, the module code looks like this:

    ```js
    import vendor_c from 'vendor_c';
    vendor_c('entry point loaded');
    ```

1. Import module

    **Static Import**

    `__webpack_require__` means `require` or `import` which we usually use. [[goto](https://github.com/tpai/how-webpack-load-modules-and-chunks/blob/master/runtime.js#L81)]

    ```js
    // Create a new module (and put it into the cache)
    var module = installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    };

    // Execute the module function
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

    return module.exports;
    ```

    **Dynamic Import**

    Webpack generate a module to store async context:

    ```js
    module_b: (function(module, __webpack_exports__, __webpack_require__) {
      /* var map = {
       *   reqModuleId: [
       *     moduleId,
       *     ...chunkIds,
       *   ],
       * };
       */
      eval(`
        var map = {
          "./dynamic_load": [
            "chunk_a",
            "chunk",
          ],
        };
        function webpackAsyncContext(req) {...}
        module.exports = webpackAsyncContext;
    ```

    Import dynamic module:

    ```js
    // webpackAsyncContext('./dynamic_load')
    __webpack_require__('module_b')('./dynamic_load');
    ```

    Inside `webpackAsyncContext('./dynamic_load')`, `__webpack_require__.e('chunk')` start a promise to load JS chunk by creating an script element and append to `document.head`,  after promise resolved return `__webpack_require__('chunk_a')`.

1. That's it!
