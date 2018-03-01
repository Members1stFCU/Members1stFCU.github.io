---
tags:
  - react
  - bootstrap
title: Configure Bootstrap with React in Webpack
author: Robyn Dunstan
---

After working through the React courses on Codecademy ([Part I](https://www.codecademy.com/learn/react-101), [Part II](https://www.codecademy.com/learn/react-102)), I wanted to set up my own development environment that includes Bootstrap and Font Awesome for styling. The online instructions I found all failed with:  

```
Error: Module '/node_modules/url/url.js' is not a loader (must have normal or pitch function
```

Since we use IE, I also set up babel to convert the JavaScript to the lowest version of IE in use here. This is how I configured all of these packages to work together. I've included the version of each package I used for reference.

## Install Node and NPM

+ If not already present, install Node.js ([Node.js website](https://nodejs.org/en/download/), [Chocolatey package](https://chocolatey.org/packages/nodejs.install)). This includes the NPM package manager.

## Configure NPM

+ Create a new folder for this project. Navigate to the folder in the Node.js command prompt, run `npm init`, and answer the prompts. 

## Install React  

The packages required to code in React are `react` and `react-dom`.

+ Run `npm install --save react` (Version 16.2.0)  
+ Run `npm install --save react-dom` (Version 16.2.0)

## Install Babel

Babel will transpile the React code into vanilla JavaScript that runs on IE.

+ Run `npm install --save-dev babel-core` (Version 6.26.0)
+ Run `npm install --save-dev babel-loader` (Version 7.1.2)
+ Run `npm install --save-dev babel-preset-react` (Version 6.24.1)
+ Run `npm install --save-dev babel-preset-env` (Version 1.6.1)

### Configure Babel

+ Create file `.babelrc` in the root directory with the following contents:

```json
{
    presets: [
        'react', [
            'env',
            {
                "targets": {
                    "browsers": ["ie 8"]
                }
            }
        ]
    ]
}
```

The order is important. We need to first convert React into modern vanilla JavaScript, and then convert the modern JavaScript into the older version that runs on IE.

## Install Webpack

Webpack takes the React code and builds it into the final version. 

+ Run `npm install --save-dev webpack` (Version 3.11.1)
+ Run `npm install --save-dev webpack-dev-server` (Version 2.11.1)
+ Run `npm install --save-dev html-webpack-plugin` (Version 2.30.1)

### Configure Webpack

+ Create file `webpack.config.js` in the root directory with the following contents:

```javascript
var HTMLWebpackPlugin = require('html-webpack-plugin');

var HTMLWebpackPluginConfig = new HTMLWebpackPlugin({
    template: __dirname + '/app/index.html',
    filename: 'index.html',
    inject: 'body'
});

module.exports = {
    entry: __dirname + '/app/index.js',
    module: {
        loaders: [{
            test: /\.js$/,
            exclude: /node_modules/,
            loader: 'babel-loader'
        }]
    },
    output: {
        filename: 'index.js',
        path: __dirname + '/build'
    },
    plugins: [HTMLWebpackPluginConfig]
};
```

The above file assumes that the source files are in subfolder `app`, with an entry point of JavaScript file `index.js` and HTML file `index.html`. The output files will be placed in subfolder `build`.

### Configure scripts

+ In file `package.json`, add two entries to the `scripts` object, as shown below:

```json
"scripts": {
    "build": "webpack",
    "start": "webpack-dev-server"
}
```

The command `npm run build` makes webpack perform its transformations from React files in the `app` subfolder to a vanilla JavaScript page in the `build` subfolder that will run in IE. 

The command `npm run start` starts a local server to display the transformed web app. It will also automatically update the web app with any code changes.
   
## Install Bootstrap

+ Install the Bootstrap packages.
    + Run `npm install --save bootstrap@3.3.7` (Version 3.3.7)
       Package bootstrap-webpack is not compatible with bootstrap 4.
    + Run `npm install --save-dev bootstrap-webpack` (Version 0.0.6)
+ Install the JQuery package.
    + Run `npm install --save jquery` (Version 3.3.1)
+ Install Less for compiling Bootstrap.
    + Run `npm install --save-dev less` (Version 3.0.0)
    + Run `npm install --save-dev less-loader` (Version 4.0.5)
+ Install Webpack loaders for use with Bootstrap files.
    + Run `npm install --save-dev css-loader` (Version 0.28.9)
    + Run `npm install --save-dev style-loader` (Version 0.20.1)
    + Run `npm install --save-dev url-loader` (Version 0.6.2)
    + Run `npm install --save-dev file-loader` (Version 1.1.6)
    + Run `npm install --save-dev imports-loader` (Version 0.7.1)
    + Run `npm install --save-dev exports-loader` (Version 0.7.0)
    + Run `npm install --save-dev extract-text-webpack-plugin` (Version 3.0.2)

### Configure Bootstrap

+ Add the following lines to the `module.exports` object, `module.loaders` array in file `webpack.config.js`. The syntax for using url-loader for fonts is different than the instructions I found online.

```javascript
{
    test: /\.(woff|woff2)(\?v=\d+\.\d+\.\d+)?$/,
    loader: 'url-loader',
    options: {
        limit: 10000,
        mimetype: 'application/font-woff',
        fallback: 'file-loader'
    }
}, {
    test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
    loader: 'url-loader',
    options: {
        limit: 10000,
        mimetype: 'application/octet-stream',
        fallback: 'file-loader'
    }
}, {
    test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
    loader: 'file-loader'
}, {
    test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
    loader: 'url-loader',
    options: {
        limit: 10000,
        mimetype: 'application/image/svg+xml',
        fallback: 'file-loader'
    }
}
```
  
## Install Font Awesome

+ Run `npm install --save font-awesome` (Version 4.7.0)
+ Run `npm install --save-dev font-awesome-webpack` (Version 0.0.5-beta.2)

### Configure Font Awesome

+ The loaders for font files have been configured in previous steps, so no edits to `webpack.config.js` are needed.

You're now ready to use Bootstrap and Font Awesome in React!
