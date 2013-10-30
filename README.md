nw-require
============

Rogerwang's node-webkit (https://github.com/rogerwang/node-webkit) lets you create native desktop apps using HTML, CSS and JavaScript, but also gives you access to all node.js modules, directly from the DOM.
When using require.js, this often leads to problems, due to the fact that both require.js and node.js have a global require function. This can be solved though by renaming node's require function to for instance nodereq before including require.js.
However, if you want to get access to a node.js module from within an AMD module, you still have to use:

```javascript
define(function(require) {

    var fs = nodereq('fs'),
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

etc. This is done by adding a check for whether the environment is node-webkit. If so, for every module it is checked whether it exists as node.js module. If so, the module is loaded. If not, require.js will try to load the module as a simple &lt;script&gt; tag.
Allmost all of the changes are done within the function req.load.

```javascript
if (isNodeWebkit) {

    // Check whether this module is a native node module
    try {
        var module = nodereq.resolve(moduleName);
        if (module) {
            // Store that this was a native node module, so we won't try to
            // load it as a requirejs module anymore.
            nativeNodeModule = true;
        }
    }
    catch (e) {
        // Ignore, because nothing needs to be done
    }

}

// If the module is a native nodejs module, handle here
if (nativeNodeModule) {

    // Push in the defQueue with no dependencies, and as callback function
    // an anonymous function is specified, which simply returns the
    // module, required by nodejs's require function. Then, call completeLoad
    // to complete the load
    context.defQueue.push([moduleName, [], function() {
        return nodereq(moduleName);
    }]);
    context.completeLoad(moduleName);

}
```

Note that using require's optimizer will probably fail (haven't tested this), however optimizing your scripts isn't really necessary when using node-webkit.
