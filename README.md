nw-require
============

Rogerwang's node-webkit (https://github.com/rogerwang/node-webkit) lets you create native desktop apps using HTML, CSS and JavaScript, but also gives you access to all node.js modules, directly from the DOM.
When using require.js, this often leads to problems, due to the fact that both require.js and node.js have a global require function.
This can be solved though by renaming node's require function to for instance nodereq before including require.js.
However, if you want to get access to a node.js module from within an AMD module, you still have to use:

```javascript
define(function(require) {

    var fs = global.require('fs'),
        $ = require('jquery'),
        Backbone = require('backbone');

});
```

Therefore I modified require.js to allow for

```javascript
define(function(require) {

    var fs = require('fs'),
        $ = require('jquery'),
        Backbone = require('backbone');

});
```

etc. This is done by adding a check for whether the environment is node-webkit. If so, for every module it is checked whether it exists as node.js module.
If so, the module is loaded. If not, require.js will try to load the module as a simple &lt;script&gt; tag.
Allmost all of the changes are done within the function req.load.

```javascript
// The function in the if statement checks whether the requested 
// module is a native node.js module (i.e. in a folder node_modules)
// This is done by trying to resolve the module using node-webkit's
// require function. If the module is not a native node.js module, then
// an error is thrown which is caught to return false.
if ((function() {
    try {
        nodereq.resolve(moduleName);
        return true;
    }
    catch (e) {
        return false;
    }
})()) {
    // Push in the defQueue with no dependencies, and as callback 
    // function an anonymous function is specified, which simply 
    // returns the module, required by nodejs's require function. 
    // Then, call completeLoad to complete the load
    context.defQueue.push([moduleName, [], function() {
        return nodereq(moduleName);
    }]);
    context.completeLoad(moduleName);
}
else {
    // Do the usual requirejs stuff
}
```

Note that using require's optimizer will fail since it won't find a dependency called 'fs' for instance.
This can be solved by specifying the native node.js modules as modules the optimizer should exclude.
Fore more info, see: https://github.com/jrburke/r.js/blob/master/build/example.build.js

requirejs version
============

The version of requirejs that was used, is version 2.1.14.
If you specifically need other versions of requirejs, I suggest you try to modify it yourself.
The only major changes are applied in the req.load function.
Minor changes are

```javascript
(function (global, nodereq) {
    // All code
}(this, typeof global === 'undefined' ? void 0 : global.require));
```

So you should be able to do this yourself.