---
layout: post
title: Salesforce.com Lightning Development Part 1 - external libraries
---

I have spent some time developing on the Lightning platform and would like to share my experiences as well as some patterns that might help your journey throught the Aura framework. As with other parts of the platform, Salesforce has provided an abundance of documentation for developing with Lightning, but the documentation is scattered across many pages with varying levels of detail and thouroughness.

I am a full-stack developer and spend most of my time building applications using Angular and React and I am usually annoyed at the the restrictions that come with "the platform".  The first annoyance that I encountered was that `helper` functions were scoped to a component.  This is generally fine, but I found myself wondering where I could put basic utility methods that I wanted to use anywhere in the application.

The good news is that [Salesforce provides guidance on how to do this](https://developer.salesforce.com/docs/atlas.en-us.lightning.meta/lightning/security_share_code.htm).  I have chose to follow a pattern similar to how Salesforce has implemented components and helper classes. I create an object with properties that are usually functions for simple utility methods.  I will showcase this using a really useful utility for logging so that `console.log()` statements can be toggled in production without chaning component code.

## Creating a Logger
First, create a new static resource named something obvious, such as `utils` or `lighting_utils`.
For this example, let's add a new `utils` method to the fake `window` object (this uses Salesforce's `LockerService` and is not the actual `window`).

```javascript
window.utils = ({
    foo: function() {
        console.log('HELLO, WORLD!');
    },
})
```

Now, this code will be available in any of our components that require this resource.

**Note:** You only need to do this once in the most top level component of a component tree.

```html
<ltng:require scripts="{!$Resource.counter}" afterScriptsLoaded="{!c.doInit}"/>

<!-- If you had an init handler, remove or comment this, as we don't want to init until our scripts load  -->
<!-- <aura:handler name="init" value="{!this}" action="{!c.doInit}"/> -->
```

If you need to use multiple resources, make sure to use the `join()` method to load all of them. 

```html
    <ltng:require scripts="{!join(',', $Resource.utils, $Resource.lodash, $Resource.momentjs)}" afterScriptsLoaded="{!c.doInit}" />
```

Now, you can call `utils.foo()` from your components and helper functions - YAY!

Now Let's look at something more useful, like creating a universal logger to get those `console.log()` calls out of your code and into our util static resource.  Let's refactor our static resource as shown below.

```javascript
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

This utils class provides some really neat benefits:
1. We can turn logging on/off from changing a static resource, which is much easier to change in production without a code change.
2. We don't need to do code cleanup before deploying to production, because we won't have any `console.log()` calls.
3. The original line numbers are preserved, so we don't lose any debugging capability  with this setup.

To use the service call any of the log/warn/error methods from a controller or helper class `utils.log('something needs to be logged', {foo: 'foo-bar', bar: 'baz'})`.

Example usage
```javascript
({
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
        })
    },
})
```
// TODO - show image
<!-- 2018-03-14-sfdc1-external-libraries-01.png -->

## Summary
// TODO