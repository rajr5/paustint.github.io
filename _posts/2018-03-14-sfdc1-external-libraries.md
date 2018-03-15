---
layout: post
title: Salesforce.com Lightning Development Part 1 - external libraries
---

### tl;dr
Use static resources to share utility javascript throughout your lightning components.  Look at the code samples to see how to build a logging utility to avoid using `console.log` in your code and easily turn logs off in production without component code changes.

I have spent some time developing on the Lightning platform and would like to share my experiences as well as some patterns that might help your journey throughout the Aura framework. As with other parts of the platform, Salesforce has provided an abundance of documentation for developing with Lightning, but the documentation is scattered across many pages with varying levels of detail and thoroughness.  Because the road was a bit bumpy for me, I would like to share the outcome of my experiences.

I am a full-stack developer and spend most of my time building applications using Angular and React (I know, most people fiercely support one and hate the other - but not me!) and I am usually annoyed at the restrictions that come with "the platform".  The first annoyance that I encountered was that `helper` functions were scoped to a component.  This is generally fine, but I found myself wondering where I could put basic utility methods that I wanted to use anywhere in my application. The good news is that [Salesforce provides guidance on how to do this](https://developer.salesforce.com/docs/atlas.en-us.lightning.meta/lightning/security_share_code.htm).

From past experiences, I know that figuring out a good logging strategy is not something that should be put off very long,
and it was my first use of a utility library.

## Creating a Logger
First, create a new static resource named something obvious, such as `utils` or `lighting_utils`.
For this example, let's add a new `utils` property to the fake `window` object (this uses Salesforce's `LockerService` and is not the actual `window` object).

```javascript
// utils.resource
window.utils = ({
    foo: function() {
        console.log('HELLO, WORLD!');
    },
})
```

After saving the static resource, this code will be available in any of our components that import this with the `<ltng:require />` tag.

**Note:** You only need to do this once in the most top level component of a component tree.

```html
<ltng:require scripts="{!$Resource.counter}" afterScriptsLoaded="{!c.doInit}"/>

<!-- If you had an init handler remove it, as we don't want to initialize until our scripts are loaded  -->
<!-- <aura:handler name="init" value="{!this}" action="{!c.doInit}"/> -->
```

If you need to use multiple resources, make sure to use the built-in `join()` method to load all of them. 

```html
<ltng:require scripts="{!join(',', $Resource.utils, $Resource.lodash, $Resource.momentjs)}" afterScriptsLoaded="{!c.doInit}" />
```

Now, you can call `utils.foo()` from your components or helper component functions - YAY!

Now Let's look at something more useful, like creating a universal logger to get those `console.log()` calls out of your code and into our util static resource.  Let's refactor our static resource as shown below.

```javascript
// utils.resource
window.utils = ({
    SHOW_LOG: true, // Set to false in production

    log:   function() { },
    warn:  function() { },
    error: function() { },

    logInit: function () {
        if(this.SHOW_LOG) {
            this.log = console.log.bind(console);
            this.warn = console.warn.bind(console);
            this.error = console.error.bind(console);
        }
    },
});

// Initialize our log based on the log level
window.utils.logInit();

// This is also a good place to put polyfills so they load automatically when the utils are imported.
// e.x.: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/find

```
Here is what the code is doing:
1. Exposes `log`, `warn`, and `error` functions, which are initially set to [NOOP](https://en.wikipedia.org/wiki/NOP) (empty function)
2. We then expose `logInit()` that will do nothing if `utils.SHOW_LOG === false` (production), or assign the appropriate console logging function to our initially NOOP `log`, `warn`, and `error` functions.
3. The reason we are using `bind` is because we want to be sure that the context that invokes our logging functions retains that context, which is the magic that allows the line numbers to be retained - this is a must-have in my book. (If you don't know how the `this` keyword works in JavaScript, you should spend some time researching it and learn about `bind` and `apply`)
4. We then call `window.utils.logInit()` to setup our logging based on the `SHOW_LOG` boolean flag.


This logging pattern provides some really nice benefits:
1. We can turn logging on/off from changing a static resource, which is much easier to change in production without a code change.
2. We don't need to do code cleanup before deploying to production, because we won't have any `console.log()` calls.
3. The original line numbers are preserved, so we don't lose any debugging capability  with this setup.
4. We can share this across any lightning component easily and have a consistent way to handle everything

To use the service, call any of the log/warn/error methods from a controller or helper class `utils.log('something needs to be logged', {foo: 'foo-bar', bar: 'baz'})`.

Example usage
```javascript
({
    // example helper function
    getQuoteLines: function(component, quoteId, quoteLineId) {
        return new Promise(function (resolve, reject) {

            var action = component.get("c.getQuoteLines");
            
            action.setCallback(this, function(results) {
                utils.log('getQuoteLines() results', results);
                if(results.getState() === 'SUCCESS') {
                    utils.log('results:', results.getReturnValue());
                    resolve(results.getReturnValue());
                } else if (results.getState() === 'ERROR') {
                    utils.log('getQuoteLines() ERROR', results.getError());
                    $A.log("Errors", results.getError());
                    reject(results.getError());
                }
            });

            $A.enqueueAction(action);
        });
    },
})
```

![logging-example-image-01]({{ site.baseurl }}/images/2018-03-14-sfdc1-external-libraries-01.png)

