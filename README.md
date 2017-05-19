# NPM Modules in Ember CLI

**Shim your way to `import AmazingModule from 'amazing-module'`**

## Overview

One of Ember CLI's greatest strengths is its thriving Ember Addon ecosystem. As a build tool, Ember CLI has made it relatively fast and easy to create and ship Ember-ready modules that can enhance and extend the capabilities of Ember applications in a number of ways.

However, one of the great pitfalls of Ember Addons within Ember CLI is that while it's incredibly easy to create Ember Classes that are available for consumption within any app seemingly by magic, it can be painfully difficult to make arbitrary Node Modules available for easy consumption within Ember CLI.

There are a number of addons that do the hard work of exposing NPM Modules in the background and making them availble via some nice wrapper Components (effectively creating a proxy API for interacting with a library), and even some addons with the noble goal of letting you import any module you want. The latter case sounds dreamy in theory, but in practice it can be just as difficult to get working, and the extra dependencies involved only add to the complexity and weight of your application.

Our team at HealthSparq has experimented with importing Node Modules into our applications and addons for a variety of different use cases, and over time we have managed to derive and distill a relatively "simple" process for bringing any Node Module we want into our Ember projects. Of course, as with any sort trick of this nature, the magic is easy when you actually know what spells to cast.

The intent of this document is to share what we've learned and help others to quickly create solutions in their apps for bringing in whatever node modules they want (in theory).

## How It Works

The overall implementation requires (or maybe just suggests) a passing understanding of how Ember CLI finds files, how files can be imported into projects, and mechanisms made available through the Ember CLI API that allow us to "move" files around to make previously inaccessible node modules suddenly visible and `import`-able into any Ember Class file.

The key ingredients are as follows:

1. Installation of your module of choice (and a cursory knowledge of what that module exports)
2. Creation of a shim file in your `vendor` folder to wrap a reference to your target Node Module in an AMD module
3. Edits to your Ember App's `ember-cli-build.js` or Ember Addon's `index.js` file:
  - Utilizing the `treesForVendor` hook to transpose node module assets into the `vendor` folder at build time
  - Utilizing `included` (for Addons) or the main `module.exports` statement (for Apps) to `app.import` your newly exposed assets
  - Using `app.import` in the same hook/statement as the previous step to use your shim to make your module available to the Ember Resolver (?)

We will go over each of these steps in detail in the next sections.

## Recipes For Magic 🔮

### Installing Your Module

This step is pretty straightforward; simply `npm install` or `yarn` your target module. For the sake of this exercise, we'll use [markdown](https://www.npmjs.com/package/markdown).

In your project, simply `npm install markdown` or `yarn markdown`.

Great, we've got an installed module!

### Creating a Module Shim

The first element of creating an `import`-able node module for our Ember project is creating a shim for the module. This shim goes in your project's `vendor` folder; you can call it whatever you want.

In this shim file, we will define an AMD module that will effectively act as a wrapper for the node module.

```javascript
// vendor/markdown-shim.js
define('markdown', [], function() {
  'use strict';

  return {
    'default': Markdown
  }
});
```

This is the simplest form of creating a module shim. It defines an AMD module whose `default` export is a reference to the default export of the node module.

However, the module you're working with may have additional exports that you would like to be able to import via destructuring. Luckily, this is as simple as declaring additional exports on the `return` object:

```javascript
// vendor/some-module-shim.js
define('totally-rad-module', [], function() {
  'use strict';

  return {
    'default': TotallyRadModule,
    'totallyRadMethod': TotallyRadModule.totallyRadMethod
  }
});
```

While this is a relatively straightforward solution, it would be tedious and impractical to manually declare every export you want to make available to your code, but luckily Javascript makes solving this problem programmatically relatively trivial:

```javascript
// vendor/some-module-shim.js
define('totally-rad-module', [], function() {
  'use strict';

  const exportKeys = Object.keys(TotallyRadModule);
  let exports = {
    'default': TotallyRadModule
  };

  exportKeys.forEach(key => {
    // No need to re-export the default
    if (key !== 'TotallyRadModule') {
      exports[key] = TotallyRadModule[key];
    }
  });

  return exports;
});
```

Depending on how your module is set up, this example may not be totally sufficient to expose individual exports. However, this should cover the most common use cases and have allowed for the code in the shim to do all of the heavy lifting for us. At this point it should be easy to determine any other customizations that need to be made to your shim.

### Updating Your Ember Project Files

Now that we've installed our module and created an AMD module wrapper, the last step is to modify our Ember project to expose the node module's files to Broccoli, load those files into memory, and finally load our AMD module wrapper shim.

**For Ember Addons:**

These changes must be placed directly into the `module.exports` statement of `index.js`.

```javascript
// index.js
module.exports = {
  // code goes here
}
```

**For Ember Apps:**

These changes must be placed inside of the callback method assigned to `module.exports` in `ember-cli-build.js`.

```javascript
// index.js
module.exports = function(defaults) {
  var app = new EmberApp(defaults, {});

  // code goes here
}
```

Lastly, make sure to `require` some modules at the top of your file so we can do our work:

```javascript
// top of index.js or ember-cli-build.js
const path = require('path');
const Funnel = require('broccoli-funnel');
const mergeTrees = require('broccoli-merge-trees');
```

#### Transponse Node Module Assets to Vendor Folder

Now that we've worked out the *where*, we can focus on the *what*.

First, we will want to define some code in the `treeForVendor` hook that will use some Node methods to resolve the path in which our module resides and transpose the files we choose in that path into the project's `vendor` folder during build time. Note that this does not alter the location of files within your system.

```javascript
treeForVendor(vendorTree) {
  // Set up a placeholder for all of our trees
  const trees = [];
  // Resolve the node module and discover the path and directory name in
  // which it resides.
  let modulePath = path.dirname(require.resolve('markdown'));

  // Pull in existing vendor tree
  // This is important to preserve the existing tree for the vendor folder
  // that Ember CLI has already created.
  if (vendorTree) {
    trees.push(vendorTree);
  }

  // This is the big show! This statement adds a new Broccoli Funnel to
  // the list of trees, transposing the files from the module path
  // we set up earlier into the destination directory (within `vendor`)
  // that we choose. If you don't define a `destDir`, the files will
  // be placed in the root of the `vendor` dir. You can also use the
  // `include` option to select specific file types.
  trees.push(new Funnel(modulePath, {
    destDir: 'markdown',
    include: [new RegExp(/\.js$/)]
  }));

  // Finally, merge the pre-existing vendor tree and our custom module
  // tree into one
  return mergeTrees(trees);
}
```

#### Import Our "New" Files From `vendor` Into The App

Remember, for Addons, the following code goes inside of the `included` hook; for Apps, this just lives after the `var app = new EmberApp()` invocation.

```javascript
const vendor = this.treePaths.vendor;

// Import our module into the app for bundling
app.import(`${vendor}/markdown/markdown.js`);

// The coup de grace! Now we import our AMD module wrapper shim to
// make our module available for `import` within our JS files:
app.import(`${vendor}/markdown-shim.js`, {
  exports: {
    Markdown: ['default']
  }
});
```

At this point, all of the work we need to do to import a module has been completed and it's ready for `import` in our JS files. It should be noted that the work involved in the previous two steps could be abstracted into some utility functions that accept a few arguments and do all of this repetetive importing for us, which could be useful if you want to use this technique for multiple Node modules in your Project.

### Import Your Module Into A Component

We can now `import` our module into a component for Markdown rendering:

```javascript
// app/components/render-markdown.js
import Component from 'ember-component';
import hbs from 'htmlbars-inline-precompile';
import Markdown from 'markdown';

export default Component.extend({
  didReceiveAttrs() {
    if (this.get('markdown')) {
      let markup = Markdown.render(this.get('markdown'));
      this.set('renderedMarkup', markup);
    }
  },

  layout: hbs`{{renderedMarkup}}`
});
```

It's a little bit of jumping through hoops to get to this point, but the payoff of this effort is clearly evident from the ease with which we are able to work with our modules in the context of Components. 

## Resources

Please see the included reference files for specific full-file examples demonstrating this technique both for addons and for apps. There's also an example of how the work to handle sideloading node modules into the app at build time can be abstracted for easier use with multiple modules.

- Shim Example (Coming Soon)
- Ember Addon Imports (Coming Soon)
- Ember App Imports (Coming Soon)
- Abstracting the Import Work (Coming Soon)