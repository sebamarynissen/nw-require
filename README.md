# nw-require

## Overview

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
etc.

Even 
```javascript
define(function(require) {

    var gui = require('nw.gui');

});
```
is possible. The magic is done by adding a check for whether the environment is node-webkit. If so, for every module it is checked whether it exists as node.js module.
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
        return nwreq(moduleName);
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

## Requiring other files

On plain nodewebkit, it is possible to call

```javascript
var util = require('./lib/util');
```
if you have a local folder called "lib" with some internal modules in it. Using
```javascript
define(function(require) {
    var util = require('./lib/util');
});
```
will not work however, only native node modules or modules from the node_modules folder can be loaded, due to the fact that requirejs normalizes the path first before passing it to the req.load() function.
As a consequence the base url etc. will be added.
To be able to require internal modules, a plugin called "lcl" was built in.
This allows for
```javascript
define(function(require) {
    var util = require('lcl!lib/util');
});
```
You may specify a root directory in the requirejs config object, such as
```javascript
requirejs.config({
    "config": {
    "lcl": {
        "root": "./lib"
    }
}
});
```
which allows for
```javascript
define(function(require) {
    var util = require('lcl!util');
});
```

## Install

Just download `nw-require.js` or install via bower using
```
bower install nw-require
```

## Conflicts

Some plugins (for instance the [require-handlebars-plugin](https://github.com/SlexAxton/require-handlebars-plugin)) use the global ```require()``` function instead of ```requirejs()``` which results in unexpected behavior, such as AssertionErrors being thrown because node.js complains that the path must be a string (and not an array, as in the requirejs syntax). Therefore, the lines

```javascript
// Some plugins, such as the require-handlebars-plugin use require() 
// insteadof requirejs, which will result in name conflicts and also 
// unexpected behavior, such as AssertionErrors and so on. Therefore,
// simulate the requirejs behavior of the require function when the
// requirejs syntax is used (i.e. require(['module'], function() {}))
// IMPORTANT NOTE: Below global.require() is used, but this is in fact
// window.require, because requirejs passes "this" into the closure, which
// is then renamed to global, overriding node.js's "global" object!
global.require = (function(nodewebkit, requirejs) {
    return function() {
        if (isArray(arguments[0])) {
            return requirejs.apply(null, arguments);
        }
        else {
            return nodewebkit.apply(null, arguments);
        }
    };
})(nwreq, req);
for (var i in req) {
    if (!global.require[i]) {
        global.require[i] = req[i];
    }
}
```

were added. This is - again - an override of the ```require()``` function, which detects what syntax is used, and calls the appropriate require function.

## Require.js version

The version of requirejs that was used, is version 2.1.14.
If you specifically need other versions of requirejs, I suggest you try to modify it yourself.
The only major changes are applied in the req.load function.
Minor changes are

```javascript
(function (global, nodereq, nwreq) {

    /*
     * ALL
     * OTHER
     * CODE
     */

    // Some plugins, such as the require-handlebars-plugin use require() 
    // instead of requirejs, which will result in name conflicts and also 
    // unexpected behavior, such as AssertionErrors and so on. Therefore,
    // simulate the requirejs behavior of the require function when the
    // requirejs syntax is used (i.e. require(['module'], function() {}))
    // IMPORTANT NOTE: Below global.require() is used, but this is in fact
    // window.require, because requirejs passes "this" into the closure, which
    // is then renamed to global, overriding node.js's "global" object!
    global.require = (function(nodewebkit, requirejs) {
        return function() {
            if (isArray(arguments[0])) {
                return requirejs.apply(null, arguments);
            }
            else {
                return nodewebkit.apply(null, arguments);
            }
        };
    })(nwreq, req);
    for (var i in req) {
        if (!global.require[i]) {
            global.require[i] = req[i];
        }
    }
    
    
}(this, typeof global === 'undefined' ? void 0 : global.require, require));
```

So you should be able to do this yourself.
