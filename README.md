# Wendigo :jack_o_lantern:
<img src="https://user-images.githubusercontent.com/5960567/41823576-2f7cb71a-7802-11e8-8714-521cb38b42c0.png" align="right" width="150px">


_by @angrykoala_    
[![npm version](https://badge.fury.io/js/wendigo.svg)](https://badge.fury.io/js/wendigo)
[![Build Status](https://travis-ci.org/angrykoala/wendigo.svg?branch=master)](https://travis-ci.org/angrykoala/wendigo)


> A proper monster for front-end automated testing

**Wendigo** (_/wɛndɪɡo/_)  is a wrapper of [Puppeteer](https://github.com/GoogleChrome/puppeteer) with the purpose of making automated testing easier and simpler. Install it with `npm install --save-dev wendigo`

Consider the following example of a test using Puppeteer:

```javascript
await page.click(".my-btn");
await page.waitForSelector("#my-modal")
const modalText = await page.evaluate(() => {
    const modalElement = document.querySelector("#my-modal");
    return modalElement.textContent;
})
assert.strictEqual(modalText, "Button Clicked");
```

Using Wendigo, the same test could be written like this:

```javascript
await browser.click(".my-btn");
await page.waitFor("#my-modal");
await browser.assert.text("#my-modal", "Button Clicked");
```

> Recommended Node 8.11.3 LTS or greater

**Contents**
* [Api](#api)
    * [Wendigo](#wendigo)
    * [Browser](#browser)
    * [Assert](#assert)
    * [Cookies](#cookies)
    * [Console](#console)
    * [LocalStorage](#localstorage)
    * [Requests](#requests)
    * [Webworkers](#webworkers)
    * [Errors](#errors)
    * [Selectors](#selectors)
    * [Injected Scripts](#injected-scripts)
* [Plugins](#plugins)
    * [Writing A Plugin](#writing-a-plugin)
* [Examples](#examples)
* [Development](#development)
* [Troubleshooting](#troubleshooting)
* [Acknowledgements](#acknowledgements)
* [License](#license)

# Api


## Wendigo
Wendigo is the main static class exported by the package. It provides the methods necessary to create browsers and disconnect from chrome, can be imported with `require('wendigo')`:

**createBrowser(settings)**   
Will create and return a [Browser](#Browser) instance. It will automatically launch and connect Puppeteer and Chrome if an instance is not running.

* _settings_ is an optional object with the settings to build the browser
    * `log: false`: If true, it will log all the console events of the browser.
    * `incognito: false`: If true, the browser will open as an incognito browser.
    * `userAgent`: If defined, the default user agent will be overridden.
    * `noSandbox`: Sets the option `--no-sandbox` when opening Puppeteer. This option will also be set if the env variable `NO_SANDBOX` is set (check [troubleshooting](#troubleshooting))
    * `timezone`: Sets the browser's timezone (e.g. `UTC`, `Asia/Tokyo`).
    * Any settings that can be passed to Puppeteer can be passed in createdBrowser, for example:
        * `headless: true`: If true, the browser will run on headless mode.
        * `slowMo: 0`: Slows the execution of commands by given number of milliseconds


Examples:
```js
const Wendigo = require('wendigo');
const browser = Wendigo.createBrowser(); // Using default options
```

```js
const Wendigo = require('wendigo');
const browser = Wendigo.createBrowser({
    headless: false,
    slowMo: 500
}); // Using options to see what's happening
```

**stop()**   
Will stop and disconnect all the browsers. It should be called after finishing all the tests.

**registerPlugin(name, plugin?, assertions?)**   
Registers a new plugin, for more information, check [Plugins](#plugins). This must be called before `createBrowser` for the plugins to work.

Optionally an object can be passed with the following options:
* `name`
* `plugin`
* `assertions`

**clearPlugins()**   
Removes all plugins from Wendigo. This will affect all newly created browsers.

## Browser
The Browser instance is and interface with the `page` class of Puppeteer.

### Attributes
**page**   
Puppeteer [page class](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#class-page), allows access to Puppeteer API if needed.

```js
await browser.page.evaluate(() => {
    document.querySelector("h1");
});
```

**loaded**   
True if the page has already opened and loaded.

### Methods
All the methods in Browser return a Promise than can easily be handled by using `async/await`.

**open(url, options?)**    
Opens the given url in the browser.

```js
await browser.open("http://localhost:8000");
```

The following options can be passed:

* `clearRequestMocks` (default: true): Clears all previous mocks in the requests module
* `viewport`: Viewport config to set when opening the browser, uses the same syntax as `setViewport`


**openFile(path, options?)**    
Opens the given file from the browser. Same options as `open` can be passed. The file will be passed by appending `file://` to the absolute path.

```js
await browser.open("static/index.html");
```

**close()**    
Close the browser, it should be called after finishing using the browser. Avoid creating a new browser before closing the previous one if possible. Having multiple open browsers will cause performance degradation in your tests.

```js
await browser.close();
```

**evaluate(cb, ...args)**    
Evaluates given callback in the browser, passing n arguments. Returns the Puppeteer's result of the evaluation.

```js
const selector = "h1";
const elementText = await browser.evaluate((s) => {
    return document.querySelector(s).textContent;
}, selector); // My Title
```

> This is a wrapper around browser.page.evaluate

**query(selector, childSelector?)**   
Queries the given css selector and returns a DOM element. If multiple elements are matched, only the first will be returned. Returns null if no element found.

```js
const element = await browser.query("h1");
```

Optionally, query supports 2 parameters, the first being a DOMElement and the selector as the second one. The query will then be performed only on the elements under the parent.

**queryAll(selector, childSelector?)**   
Returns an array with all the DOM elements that match the given css selector.

```js
const elements = await browser.queryAll("h1");
elements.length; // 2
```

Optionally, queryAll supports 2 parameters, the first being a DOMElement and the selector as the second one. The query will then be performed only on the elements under the parent.

> All the Dom elements returned by queryElement and queryAll can be used instead of a selector in other methods and assertions.

**queryXPath(xPathSelector)**   
Returns an array with the DOM elements matching the xPath selector.

```js
const elements = await browser.queryXPath('//p[contains(text(),"My first paragraph")]');
elements[0].textContent; // "My first paragraph"
```

> The DomElement class returned by all query methods provides an interface to Puppeteer's ElementHandle class, it can be accesed with the property `element`

**addScript(scriptPath)**   
Executes the given script in the browser context. Useful to set helper methods and functions. This method must be called after the page is already loaded, if another page is loaded, the scripts won't be re-executed. If these scripts are required for a plugin to work, remember to execute this method on the `_afterOpen` hook.

It is heavily recommended to only use this to load helper functions, and not execute anything that might cause side effects. Anything loaded as a script may interfere with the behavior of the page or Wendigo. It is recommended to **always** check if the object of function you are loading already exists before loading, remember that `WendigoUtils` and `WendigoQuery` objects in `window` are required for Wendigo to work, so do not override them.

**class(selector)**    
Returns and array with the classes of the first element returned from the given css selector. Throws if no element is found.

```js
const classes = await browser.class("div.container.main"); // Returns ["container", "main", "another-class"]
```

Using a dom node:
```js
const node = await browser.query("div.container.main");
const classes = await browser.class(node); // Returns ["container", "main", "another-class"]
```

**value(selector)**   
Returns the value of the first element with given selector. Returns _null_ if no element or value found.

```js
const value = await browser.value("input.my-input");
```

**attribute(selector, attributeName)**   
Return the attribute value of the first element found with given selector. Throws if no element is found. Returns `""` if the attribute is set but no value is given and `null` if the attribute doesn't exists.

```js
const classAttribute = await browser.attribute(".my-element", "class"); // Returns "my-element another-class"

const hiddentAttr = await browser.attribute(".my-hidden-element", "hidden"); // Returns ""
const hiddentAttr2 = await browser.attribute(".not-hidden-element", "hidden"); // Returns null
```

**styles(selector)**    
Returns an object with all the computed css styles of the first element matching the given selector.

```js
const styles = await browser.styles("h1.my-title");
styles.color; // 'rgb(255, 0, 0)'
```

**style(selector, style)**    
Returns the value of the given style of the first element matching the give nselector. Returns undefined if the style doesn't exists. Throws if the element is not found.

```js
const style = await browser.style("h1.my-title", color); // 'rgb(255, 0, 0)'
```

**checked(selector)**   
Returns true if the first element matching the given selector (checkbox) is checked. If the value is not a checkbox and doesn't have checked property set, it will return undefined.


**text(selector)**   
Returns an array with all text contents of the elements matching the css selector

```js
const texts = await browser.text("p"); // ["My First Paragraph", "My Second Paragraph"]
```

**click(selector, index)**   
Clicks all the elements with the matching css selector, if the index parameter is set, only the nth element will be clicked. Returns the number of elements clicked.

```js
await browser.click("button.btn");
```

**clickText(selector?, text, index?)**   
Clicks all the elements matching given text. Returns the number of elements clicked.

```js
await browser.clickText("Click Me!");
```

Optionally a selector can be passed as first argument to only click elements under the given selector. If an index is passed, only the nth element found will be clicked, be aware of the type passed down to index if selector is not passed:

```js
await browser.clickText("Click Me!", 2); // Clicks the second element
await browser.clickText("Click Me!", "2"); // Will search for an element with selector "Click Me!" and text "2"
await browser.clickText(".container", "Click Me!", 2); // Clicks the second element with given text under the element ".container"
```

**clickAndWaitForNavigation(selector, timeout=500)**   
Clicks an element and waits until a navigation event is triggered. Recommended for links to different pages. Keep in mind that not all the clicks will trigger a navigation event.

```js
await browser.url(); // my-page/account
await browser.clickInWaitForNavigation(".home-button");
await browser.url(); // my-page/home
```

> clickAndWaitForNavigation may delay up to 100ms after the given timeout while waiting for the page to load

**check(selector)**    
Checks the first element matching given selector. Setting its checked property to true.

**uncheck(selector)**    
Unchecks the first element matching given selector. Setting its checked property to false.

**title()**   
Returns the page title.

**html()**   
Returns the page html as string. It will return the html as it was before performing any actions.

**frames()**  
Returns all the [frames](https://github.com/GoogleChrome/puppeteer/blob/v1.8.0/docs/api.md#class-frame) attached to the page

**url()**   
Returns the current url of the page

**wait(ms=250)**   
Waits for the given milliseconds.

**waitFor(selector, timeout=500, ...args?)**   
Waits for given selector to exists and be visible, with the given timeout in milliseconds.

```js
await browser.waitFor(".popup");
```

If a function is passed instead of a selector, it will wait for that function to resolve in the browser context to true, the optional arguments are passed to the function.

```js
await browser.waitFor((s) => { // Waits for 2 or more elements to be in the page
    const docs = document.querySelectorAll(s);
    return docs.length > 2;
}, 600, ".my-elements");
```

> Css and Xpath selectors supported

**waitUntilNotVisible(selector, timeout=500)**   
Waits until the given selector is no longer visible or doesn't exists, with the given timeout in milliseconds.

```js
await browser.waitUntilNotVisible(".toast");
```

**waitForUrl(url, timeout=500)**    
Waits for the page to have the given url.

```js
await browser.click("a");
await browser.waitForUrl("my-url");
```

**waitForRequest(url, timeout=500)**    
Waits until a request with given url is done. This will resolve immediately if the requests was already made, to wait without taking in account past requests use `waitForNextRequest`.

```js
await browser.waitForRequest("my-url");
```

**waitForResponse(url, timeout=500)**    
Waits until a response to the given url is done. This will resolve immediately if the response was already received, to wait without taking in account past requests use `waitForNextResponse`.

**waitForNextRequest(url ,timeout=500)**   
Waits until next request with given url is done. If the request was already made, this method will wait until next one.

**waitForNextResponse(url ,timeout=500)**   
Waits until next response with given url is received. If the response was already received, this method will wait until next one.

**waitForNavigation(timeout=500)**   
Waits until next page is loaded, recommended after following a link to a different page. Keep in mind that a navigation within a SPA won't necessarily trigger a navigation event.

> waitForNavigation may delay up to 100ms after the given timeout while waiting for the page to load

**findByText(selector?, text)**   
Returns an array with the elements with text content matching the given text.  

```js
const elements = await browser.findByText("My First Paragraph");
elements.length; // 1
```

Optionally, a selector can be passed as first argument to perform a text search on children of that element only.

**findByTextContaining(selector?, text)**    
Returns an array with all the elements with a text that contains the given text.

```js
const elements = await browser.findByTextContaining("Paragraph");
elements.length; // 2
```

Otionally, a selector can be passed as first argument to perform a text search on children of that element only.

**type(selector, text)**   
Types given text in the first element matching given selector. If a value is already present, writes the new value at the beginning.


```js
await browser.type("input.my-input", "My Input");
```

**keyPress(key, count?)**    
Press a keyboard key, the key can be the name of any key supporter by [Puppeteer](https://github.com/GoogleChrome/puppeteer/blob/master/lib/USKeyboardLayout.js)

If an array is passed, all the keys will be pressed consecutively. If count parameter is passed, all the keys will be pressed that given count times.

```js
await browser.keyPress("Enter"); // Press Enter once
await browser.keyPress("Escape", 2); // Press Enter twice
```

**uploadFile(selector, path)**   
Sets the value of an input file element matching given selector. Path can be absolute or relative to the current working directory.


```js
await browser.uploadFile("input.my-file-input", "./my/file/path.txt");
```

**select(selector, value)**   
Will select the given value in the _select_ tag of the first element matching the given selector, removing all previous selections. Returns an array with the values that could be selected correctly.

Value can be a string or an array. If the select is multiple all elements in value will be selected, if not only the first element in the select options will.

Will throw if no elements were found.

```js
await browser.select("select.language-select", ["spanish", "english"]); // Returns ["spanish", "english"]
```

If the option doesn't have a value, the text should be provided.

> Only Css Selectors supported

**clearValue(selector)**   
Clears any value that exists in any of the elements matched by the given selector. Setting the value to "".

```js
await browser.clearValue("input.my-input");
```

**innerHtml(selector)**    
Returns an array with the innerHtml strings of all the elements matching the given selector

```js
await browser.innerHtml("p"); // ["my <b>first</b> paragraph"]
```

> Css, Xpath and Dom selectors supported

**setValue(selector, value)**    
Sets the given value on all the elements matching the given selector. Returns the number of elements changed, throws if no element found.
```js
await browser.setValue("input", "new val"); // Returns 1
await browser.assert.value("input", "new val");
```
This method won't trigger certain events, use `type` and `select` when possible.

> Css, Xpath and Dom selectors supported

**options(selector)**    
Returns the selector options values of the first element matching the given selector. Throws if no element is found. If the element doesn't have options (i.e. is not a selector) an empty array is returned.

```js
const options = await browser.options("selector.my-selector"); // ["value1", "value2"]
```

> Css, Xpath and Dom selectors supported

**selectedOptions(selector)**    
Returns all the selected options of the first element matching the given selector. If no value is set, the text of the option will be returned.

Will throw if no element is found.

> Css, Xpath and Dom selectors supported

**back()**     
Navigates to previous page in history.

**forward()**     
Navigates to next page in history.


**refresh()**     
Reloads current page.

**setViewport(viewportConfig)**     
Sets the configuration of the page viewport, using the same config as [Puppeteer method](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#pagesetviewportviewport).

```js
await browser.setViewport({width: 300});
```

> Unlike Puppeteer setViewport, no parameter is required, as the current values will be used for the new viewport.

**waitForPageLoad()**    
Waits until a dom ready event is fired, this method will also wait until Wendigo is ready to perform assertions on the given page.

**focus(selector)**    
Focus the first element matching the given selector.

> Only CSS selectors supported

**hover(selector)**    
Hovers over the first element matching the given selector.

> Only CSS selectors supported

**scroll(value, xValue?)**    
Vertically scrolls the page to the given value on pixels, an optional xValue can be passed for horizontal scrolling. If value is a selector or DomElement, the page will scroll until that element is at view.

> Css, Xpath and Dom selectors supported

## Assert
The submodule `browser.assert` provide some out-of-the-box assertions that can be used to easily write tests that are readable without having to specifically query for elements o perform evaluations. All the assertions have a last optional parameter (msg?) to define a custom assertion message.

**exists(selector, msg?)**   
Asserts that at least one element with given css exists

```js
await browser.assert.exists("h1.main-title");
```

**visible(selector, msg?)**   
Asserts that the at least one element matching the selector is visible.

An element will considered visible if:
* Exists
* The computed style doesn't contain display: none or visibility: hidden
* All the parents are visible

**text(selector, expected, msg?)**   
Asserts that at least one element matching the given selector has the expected string or regex.
If expected is an array, all texts in it should match.

```js
await browser.assert.text("p", "My First Paragraph");
```

**textContains(selector, expected, msg?)**   
Asserts that at least one element matching the given selector contains the expected text.

```js
await browser.assert.textContains("p", "My First");
```

**title(expected, msg?)**   
Asserts that the page title matches the expected string or regex.

**class(selector, expected, msg?)**   
Asserts that the first element matching the selector contains the expected class.

```js
await browser.assert.class("div.container.main-div", "container");
```

**url(expected, msg?)**   
Asserts that the current url matches the given string or RegExp.

**value(selector, expected, msg?)**   
Asserts that the first element matching the selector has the expected value.

```js
await browser.type("input.my-input", "Dont Panic");
await browser.assert.value("input.my-input", "Dont Panic");
```

**element(selector, msg?)**   
Asserts that exactly one element matches given selector. Same as `elements(selector, 1)`.

**elements(selector, count, msg?)**   
Asserts the number of element that matches given selector.

The count parameter can be a number of the exact number of elements expected or an object with the following properties:    
    * _atLeast_: Expects at least the given number of elements.
    * _atMost_: Expects up to the given number of elements.
    * _equal_: Expects the exact number of elements.

```html
<p>Paragraph 1</p>
<p class="second">Paragraph 2</p>
```

```js
await browser.assert.elements("p", 2); // Ok
await browser.assert.elements("p", {equal: 2}); // Ok
await browser.assert.elements("p", {atLeast: 1, atMost:3}); // Ok
await browser.assert.elements("p.first", 0); //Ok

await browser.assert.elements("p.second", 2); // Fails
await browser.assert.elements("p.second", {atLeast: 1}); // Ok
```

**attribute(selector, attribute, expected?, msg?)**   
Asserts that at least one element element matching the given selector contains an attribute matching the expected value. If no expected value is given, any not null value for the attribute will pass. The expected value can be a string or regex

```js
await browser.assert.attribute(".hidden-class", "class", "hidden-class");
await browser.assert.attribute(".hidden-class", "hidden");
```

To pass a custom message without specifying an expected value, you can pass `undefined`:
```js
await browser.assert.attribute(".hidden-class", "hidden", undefined, "hidden-class doesn't have attribute hidden");
```

You can check an attribute doesn't exists passing `null` as expected argument or using `assert.not.attribute`

If the element doesn't exists, the assertion will fail.

**style(selector, style, expected, msg?)**   
Asserts that the first element matching the given selector has an style with the expected value. The assertion will throw an error if no element is found.

```js
await browser.assert.style("h1", "color", "rgb(0, 0, 0)");
```

**href(selector, expected, msg?)**   
Asserts that the any matching the given selector contains an attribute href with expected value.

```js
browser.assert.href("a", "foo.html");
browser.assert.href("link", "styles.css");
```

> Same as `browser.assert.attribute(selector, "href", expected, msg?)`

**innerHtml(selector, expected, msg?)**    
Asserts that at least one element matching the given selector has the expected innerHtml.
The expected html can be either a _string_ or a _Regex_ value.

The assertion will throw if no element is found.

```js
await browser.assert.innerHtml("p", "my <b>first</b> paragraph");
```

> Css, Xpath and Dom selectors supported

**options(selector, expected, msg?)**    
Assets that the first element with given selector has the expected options value. Expected can be a string, if only one option is given, or an array if multiple options are given. All expected options must match in the same order.

```js
await browser.assert.options("select.my-select", ["value1", "value2"]);
```

> Css, Xpath and Dom selectors supported

**selectedOptions(selector, expected, msg?)**   
Assert that the first element with given selector has the expected options selected. Expected can be a string, if only one option is given or an array. All the selected options must match the expected options in the same order.

> Css, Xpath and Dom selectors supported

**global(key, value?, msg?)**    
Asserts that the global object (window) has the given key with the expected value. If not value (or undefined value) is provided, it will assert that the key exists with a not undefined value.

```js
browser.assert.global("localStorage");
browser.assert.global("my-val", "dontpanic");
```

> Assertions related to LocalStorage can be found under each section

**cookie(name, expected?, msg?)**   
Asserts that the cookie with the given name exists. If the expected parameter is passed, it will check that the cookie has that value.

```js
browser.assert.cookies("username");
browser.assert.cookies("username", "arthur_dent");
```

**checked(selector, msg?)**   
Asserts that the first element matching the given selector has a checked value set to true.

> Css, Xpath and Dom selectors supported

**disabled(selector, msg?)**    
Asserts that the first element matching the given selector is disabled (has attribute disabled).

> Css, Xpath and Dom selectors supported

**enabled(selector, msg?)**    
Asserts that the first element matching the given selector is enabled (doesn't have attribute disabled).

> Css, Xpath and Dom selectors supported

**focus(selector, msg?)**    
Asserts that an element matching the given selector is focused.

```js
browser.click(".btn");
browser.assert.focus(".btn");
```

> Css, Xpath and Dom selectors supported

**redirect(msg?)**    
Asserts that the opened url is a redirection.

**console(options, count?, msg?)**    
Assets that at least one console event with given options exists, if count is set, asserts that the exact number of events exists. The options can be:
* `text`: Asserts for the console event to have a text matching the given string or regex
* `type`: Asserts that the event is of the given type (log, info, error,...)

```js
await browser.assert.console({
    text: "Hello World!",
    type: browser.console.LogType.log
});
```

**webworker(options, msg?)**    
Assert that at least one webworker is running, the following options can be passes:
* `url`: Matches only the webworkers with given url
* `count`: Matches exactly the given number of webworkers running.

```js
await browser.assert.webworkers({url: "foo.js"}); // At least one webworker with given url running
await browser.assert.webworkers(); // at least one webworker running
await browser.assert.webworkers({count: 0}); // No webworkers running
```

### Negative assertions
Most of the browser assertions have a negative version that can be used with `browser.assert.not`. Most of the "not" assertions are simply the inverse of the positive version.

**not.exists(selector, msg?)**   
Asserts that no element matching given selector exists.

```js
await browser.not.exists("h1.foo.bar");
```

**not.visible(selector, msg?)**   
Asserts that no elements with given selector is visible. If no element matches, it will be considered as not visible as well.

**not.text(selector, expected, msg?)**   
Asserts that no element matching the given selector matches the expected text.
If expected is an array, no element in it should match any element with given selector

```js
await browser.assert.not.text("p", "This text doesn't exists");
```

**not.textContains(selector, expected, msg?)**   
Asserts that no elements matching the given selector contain the expected text.

```js
await browser.assert.not.textContains("p", "doesn't exis");
```

**not.title(expected, msg?)**   
Asserts that the title of the page is not the expected string.

**not.class(selector, expected, msg?)**   
Asserts that the first element matching the selector doesn't contain the expected class. It will throw if the element is not found.

**not.url(expected, msgs)**   
Asserts that the url of the page doesn't match the expected string.

**not.value(selector, expected, msg?)**    
Asserts that the first element with the given selector doesn't have the expected value.

**not.element(selector, msg?)**    
Asserts that the number of elements matching the given selector is 0.

**not.attribute(selector, attribute, expected?, msg?)**    
Asserts that no element matching the given selector doesn't contain an attribute with the expected value. If no expected value is given, any not null value on the attribute will fail.

```js
await browser.assert.not.attribute(".not-hidden-class", "class", "hidden-class");
await browser.assert.not.attribute(".not-hidden-class", "hidden");
```

To pass a custom message without specifying an expected value, you can pass undefined:
```js
await browser.assert.not.attribute(".hidden-class", "href", undefined, "hidden-class has attribute href");
```
If the element doesn't exists, the assertion will fail.

> Keep in mind that passing null as expected value will assert that the attribute exists and it is not recommended

**not.style(selector, style, expected, msg?)**   
Asserts the first element matching the selector doesn't has a style with given value.

**not.href(selector, expected, msg?)**   
Asserts that no element matching the given selector doesn't contain an attribute href with the expected value.

> Same as `browser.assert.not.attribute(selector, "href", expected, msg?)`

**not.innerHtml(selector, expected, msg?)**    
Asserts that at least no element matching the given selector has the expected innerHtml.
The expected html can be either a _string_ or a _Regex_ value.

The assertion will throw if no element is found.

```js
await browser.assert.not.innerHtml("p", "not <b>a</b> paragraph");
```

> Css, Xpath and Dom selectors supported

**not.selectedOptions(selector, expected, msg?)**   
Assert that the first element with given selector doesn't have the expected options selected. Expected can be a string, if only one option is given or an array. The assertion will only fail if all the expected options match the selected options in the same order.

> Css, Xpath and Dom selectors supported

**not.global(key, value?, msg?)**    
Asserts that the global object (window) doesn't have the given key with the expected value. If not value (or undefined value) is provided, it will assert that the key doesn't exist or it is undefined.

**not.cookie(name, expected?, msg?)**    
Asserts that the cookie with given name doesn't have the expected value. If no expected value is passed, it will check that the cookie doesn't exists (is undefined).

```js
browser.assert.not.cookies("not-a-cookie");
browser.assert.not.cookies("username", "not-user");
```

> Assertions related to LocalStorage can be found under each section


**not.checked(selector, msg?)**   
Asserts that the first element matching the given selector has a checked value set to false.

Note that if the element doesn't have a checked value (i.e. is not a checkbox) this assertion will throw.

> Css, Xpath and Dom selectors supported

**not.disabled(selector, msg?)**    
Asserts that the first element matching the given selector is not disabled (same as assert.enabled).

> Css, Xpath and Dom selectors supported

**not.enabled(selector, msg?)**    
Asserts that the first element matching the given selector is not enabled (same as assert.disabled).

> Css, Xpath and Dom selectors supported

**not.focus(selector, msg?)**    
Asserts that none of the elements matching the given selector is focused.

> Css, Xpath and Dom selectors supported

**not.redirect(msg?)**    
Asserts that the current opened page is not a redirection.


## Cookies
The module `browser.cookies` provides a way to easily handle cookies through Puppeteer's api. All methods return Promises.

**all()**    
Returns all the cookies in the current page as an object

```js
const cookies = await browser.cookies.all(); // {username: "arthur_dent", email: "arthur@dent.com"}
```

**get(name)**    
Returns the value of the cookie with given name. Returns undefined if the cookie doesn't exists

```js
const cookie = await browser.cookies.get("username"); // "arthur_dent"
```

**set(name, value)**    
Sets the value of the cookie with given name. If it already exists it will be replaced.

```js
await browser.cookies.set("username", "marvin");
```

**delete(name)**   
Deletes the cookie with given name if exists. Optionally an array can be passed and all the cookies will be removed. Won't do anything if the cookie doesn't exists.

```js
await browser.cookies.delete("username");
await browser.cookies.delete(["username", "email"]);
```

**clear()**    
Deletes all the cookies of the current page.

```js
await browser.cookies.clear()
```

## Console
The `browser.console` provides a list of the logs generated by the current page (a.k.a console.* methods). The logs are given as a custom class [Log](#log) Note that none of these methods requires the user of async/await.

**all()**    
Returns an array with all the logs generated by the page.

```js
const logs = browser.console.all();
logs[0].text; // "Hello World!"
```

**clear()**    
Clear all the current logs. Note that logs are cleared when `browser.close()` is called, but not when a new page is opened.


**filter(options)**    
Returns an array with all the logs matching the given parameters, options can be:
* `type`: The log type (`log`, `info`, `error`), it must be a string matching [Puppeteer's Console Types](https://pptr.dev/#?product=Puppeteer&version=v1.5.0&show=api-consolemessagetype), some of the most common types are accessible through `browser.console.LogTypes`.
* `text` a string or regex matching the log output

If no options are passed, all the logs will be returned, the options can be used together.

```js
const errorLogs = browser.console.filter({type:browser.console.LogTypes.Error});
errorLogs[0].text; // "Oh No! An Error"
```

```js
const logs = browser.console.filter({text: /Hello/});
logs[0].text; // "Hello World!"
```

### Log
The class Log provides an abstraction over Puppeteer's [ConsoleMessage class](https://pptr.dev/#?product=Puppeteer&version=v1.5.0&show=api-class-consolemessage). Log has the following attributes:

* _text_: The Log text, if multiple parameters were passed to console method, these will be concatenated with spaces
* _type_: The Log type (string)

> The core Puppeteer class is also accessible through myLog.message

### LogTypes
All the log types are strings, but some of the most common types are accessible in `browser.console.LogTypes`. For more possible types, check [Puppeteer Docs](https://pptr.dev/#?product=Puppeteer&version=v1.5.0&show=api-consolemessagetype):

* _log_
* _debug_
* _info_
* _error_
* _warning_
* _trace_

## LocalStorage
The module `browser.localStorage` provides a simple wrapper around the browser localStorage. All the methods return Promises.

**getItem(key)**    
Returns the item with the given key. If no item exists return null.

```js
const value = await browser.localStorage.getItem("my-key"); // returns my-value
```

**setItem(key, value)**    
Sets the given key with the given value.

```js
await browser.localStorage.setItem("my-key", "my-value");
```

**removeItem(key)**   
Removes the item with given key.

**clear()**   
Removes all the items on the store.

**length()**   
Returns the number of items in the store.

```js
const itemsLength = await browser.localStorage.length(); // 3
```

### LocalStorage Assertions
Assertions related local storage can be accessed through `browser.assert.localStorage`.

**exist(key, msg?)**    
Asserts that the item with given key exists in the localStorage (i.e. not null).

```js
browser.assert.localStorage.exist("my-key");
```

Alternatively, if an array is passed as key, all the elements will be checked.

**value(key, expected, msg?)**    
Asserts that the item with given key has the expected value.

```js
browser.assert.localStorage.value("arthur", "dontpanic");
```

Alternatively, an object can be passed instead of key/expected with the elements that should be checked:

```js
browser.assert.localStorage.value({arthur: "dontpanic", marvin:"the paranoid"});
```

**length(expected, msg?)**    
Asserts that the localStorage has the expected length.

**empty(msg?)**    
Asserts that the localStorage is empty (i.e. length>0)

> All these assertions have the negative `browser.assert.localStorage.not`.

## Requests
The Requests module allows to get and filter the requests made by the browser since the page was opened.

> All the requests objects are [Puppeteer's Requests](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#class-request)

**filter**    
Returns a filter over the requests. Check [Filtering Requests](#filtering-requests) for examples.

**all**    
Returns all requests, ordered by when it was dispatched.
```js
await browser.requests.all;
```

**mock(url, options)**    
Mocks all the requests to the given url, sending the given response instead. If a method (`GET`, `POST`...) is specified, only requests to given method will be mocked. The url can be a full url string (`http://...`) or a regex.

The following options are supported:

* `status` Response status code, defaults to 200.
* `headers` Optional response headers.
* `contentType` If set, equals to setting Content-Type response header.
* `body` Optional response body. It can be a string or a json-serializable object
* `delay` Optional delay to wait for the response to be fullfilled, in ms
* `auto` if set to false, the request won't be fullfilled automatically and a manual trigger must be defined,default to true
* `method` defines the method (`GET`, `POST`, ...) to mock
* `queryString`: If set, only requests with the exact query string will be mocked, accepts string or object
    * By default, all requests with the given url, regardless of the query string will be mocked, unless a querystring is set in the url or in the options.
* `redirectTo`: If set, the mock will return the response of the given url instead of the original call, maintaining the query string, keep in mind that the redirected request won't trigger any mocks. E.g. `request.mock("http://localhost:8010", {redirectTo: "http://localhost:9010"})` will change the port where all request in the page are sent.


> This object properties will be used with the interface of Puppeteer's [respond method](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#requestrespondresponse)

> If multiple mocks match a requests, the more specific will be used.

```js
// All requests made to /api will return 200 with the given body
browser.requests.mock("http://localhost:8000/api", {
    body: `{result: "ok"}`
});
```

Mock will return a RequestMock object, with the following properties:

* `called`: If the mock has been called.
* `timesCalled`: The times the mock has been called.
* `response` : The response the mock is returning (read only).
* `url`: Mocked url.
* `queryString`: The mock queryString.
* `immediate`: If the mock will return immediately (delay=0).
* `waitUntilCalled(timeout=500)`: Waits until the mock is called .
* `auto`: If the request will be completed automatically.

```js
const mock = browser.requests.mock("http://localhost:8000/api", {
    body: {result: "ok"}
});
mock.called; // false
mock.timesCalled; // 0
callApi(); //  { result: "ok" }
mock.called; // true
mock.timesCalled; // true
```

The mock will also provide an assertion interface in `mock.assert` with the following assertions:

* `called(times?, msg?)`: asserts that the mock has been called the given number of times, if times parameter is not given, the assertion will throw if no calls were made.
* `postBody(expected, msg?)`: asserts that the mock has been called with the given body in the request, expected can be an string, object, or RegExp.

```js
const mock = browser.requests.mock("http://localhost:8000/api", {
    body: {result: "ok"}
});
mock.assert.called(0);
callApi("my request"); // POST requests with given body
mock.assert.called(0);
mock.assert.postBody("my request");
```


All mocks are removed when opening a different page with `browser.open` unless the option `clearRequestMocks` is set to false.

If the mock is not auto, it can be manually triggered with the method `trigger()`, this method cannot be called with auto mocks:

```js

const mock = browser.requests.mock("http://localhost:8000/api", {
    body: {result: "ok"},
    auto: false
});
callApi();
mock.trigger();
```

**removeMock(url, options?)**    
Removes the mock with the given url. If the original mock has a method or queryString, these must be provided in options.

**clearRequests()**    
Clears the list of requests.

**clearMocks()**    
Remove all the request mocks.

### Filtering Requests
To filter the requests made by the browser, you can use `browser.request.filter`.

For example, to filter requests with status code of 200:

```js
const filteredRequests = await browser.requests.filter.status(200).requests;
```

The available filters are:

**url(value)**    
Filters by the given url. The url can be a string or a regex.

```js
await browser.requests.filter.url("http://localhost:8002/api").requests;
```

**method(value)**    
Filters by request method (`GET`, `POST`,...)

**status(value)**    
Filters by response status (`200`, `400`)

**fromCache(value=true)**    
Filters whether the response comes from the browser cache or not.

**responseHeaders(headers)**   
Filters requests where the response has all the given headers with the given values. The expected value can be a string or regex.

```js
await browser.requests.filter.responseHeaders({
    'content-type': /html/,
})
```

**ok(isOk=true)**    
Filters request which response are considered successfull (status is between 200 and 299).


Filters can be joined to perform a filter of several fields.

```js
//Filters all the POST requests made to any url with api that are not cached and returned a success code
await browser.filter.url(/api/).method("POST").ok().fromCache(false).requests;
```

**postBody(expected)**    
Filters requests by post body, the body can be a String, Object or regex.

```js
// Filters all DELETE requests made to with json body
await browser.filter.url(/api/).method("DELETE").body({id: 5}).requests;
```

**responseBody(expected)**      
Filters requests by response body, the body can be a String, Object or regex.

```js
const byResponseFilter = await browser.requests.filter.url(/api/).responseBody({response: 'OK'}).requests;
```

> Keep in mind that some filters like status require the requests to be finished. Use `await browser.waitForResponse()` before filtering to make sure the requests was completed.

### Requests Assertions
Assertions related requests can be accessed through `browser.assert.request`. Note that in assertions, request is singular.

Like filters, request assertion don't need `await` and can be concatenated. All the assertions will check that at least one request with the given constraints was made.

**url(expected, msg?)**    
Asserts that at least one request is made to the given url. The url can be a string or regex.

```js
await browser.assert.request.url(/api/);
```

**method(expected, msg?)**    
Asserts that at least one request was made with the given method (`GET`, `POST`, ...).

```js
await browser.assert.request.method("GET");
```

**status(expected, msg?)**    
Asserts that a response was received with the given status.

```js
await browser.assert.request.status(200);
```

> Note that this method requires the request to be finished.

**responseHeaders(expected, msg?)**    
Asserts that a response was received with the given headers. The expected variable is an object with one or more key values representing the expected headers. The value can be either a string or regex.

```js
await browser.assert.request.responseHeaders({
    'content-type': /html/,
})
```

**ok(expected=true, msg?)**    
Asserts that an successful response was received (status is between 200 and 299), or false if false is given.


**postBody(expected, msg?)**    
Asserts that a request contains the given post body (regardless of method). The expected value can be a string, regex or object.

```js
await browser.assert.request.postBody({status: "OK"});
```

**responseBody(expected, msg?)**    
Asserts that a request response contains the given body. The expected value can be a string, regex or object.

```js
await browser.assert.request.responseBody({response: "OK"});
```

**exactly(expected, msg?)**     
Asserts that the exact given number of requests match the assertions. Expected can be any positive number or 0.

```js
await browser.assert.request.url("localhost:800/api"); // asserts that at least one request is made to given url
await browser.assert.request.url("localhost:800/api").exactly(2); // asserts that 2 requests are made to given url
await browser.assert.request.url("localhost:800/api").exactly(0); // asserts that no requests are made to given url
```



Concatenating multiple assertions is possible:

```js
// Asserts that a POST method was done to the api endpoint
await browser.assert.request.method("POST").url("localhost:8000/api");
```

> Negative assertions are not supported for requests

## Webworkers
The webworkers module allows to retrieve all the webworkers in the current page:

**all()**    
Returns all the webworkers currently executing in the page. Each webworker will have the following properties:

* _url_: Returns the webworker file url
* _worker_: Returns the [Puppeteer's Worker instance](https://pptr.dev/#?product=Puppeteer&version=v1.5.0&show=api-class-worker)


## Errors
Wendigo errors can be accessed through `Wendigo.Errors`. These Errors will be thrown by Wendigo browser:

**AssertionError**   
Extends from Node.js Assertion Error. It will be throw for any assertion.

**QueryError**    
Error defining a problem with a DOM query. Generally Thrown as an unexpected result of a query made in an action or assertion.

**TimeoutError**
Timeout error, it will be thrown in waitFor methods. Keep in mind that this error is **not** the same as [Puppeteer's TimeoutError](https://pptr.dev/#?product=Puppeteer&show=api-class-timeouterror)

**FatalError**    
Defines a Fatal Error with Puppeteer (e.g. a connection error)


## Selectors
Most Wendigo methods and assertions will require a selector to localize the element in the DOM, unless specified, any method will accept 3 different kind of selectors:

* **css**: Such as `#my-id` or `.container`, any selector supported by the standard `document.querySelector`.
* **xpath**: The standard [XML Path Language](https://en.wikipedia.org/wiki/XPath) allowing more complex queries.
* **DomElement**: The result of `browser.query` can be directly used as a selector.

## Injected Scripts
For Wendigo to work properly, it must inject some scripts into the web page within the browser's context at runtime. Usually these scripts will only be used by Wendigo, but you can still access them when using `evaluate` in your code or when writing a plugin.

### WendigoUtils
Wendigo Utils contain several methods and utilities for wendigo, it can be accessed in the browser's context (in an `evaluate` callback) through the global variable `WendigoUtils` or `window.WendigoUtils`. The following methods are exposed:

* **isVisible(element)**: Returns true if an element is visible.
* **queryElement(selector)**: Returns the first element matching the given selector. The selector can be css, xpath or an element.
* **queryAll(selector)**: Returns all the elements matching the given selector (css, xpath or dom element).
* **xPathQuery(xPath)**: Returns all the elements matching the given xPath selector.
* **getStyles(element)**: Returns all the styles of the given element.

### WendigoQuery
The variable `WendigoQuery` or `window.WendigoQuery` exposes several utilities regarding Wendigo querying system, these shouldn't be used by user's code or plugins as `WendigoUtils` already exposes the methods to perform these queries.

# Plugins
Wendigo supports plugins to extends its capabilities with custom features and assertions. These plugins can be added with `registerPlugin`

[**wendigo-vue-plugin**](https://github.com/angrykoala/wendigo-vue-plugin)   
This plugin support several methods and assertions to use along with pages using [Vue](https://vuejs.org).


## Writing A Plugin
To write a plugin you must write classes defining the new methods and then registering them in Wendigo with `registerPlugin`

```js
class MyPlugin {
    constructor(browser) { // The plugin will receive the browser instance in the constructor
        this._browser = browser;
    }

    getHeaderTitle() { // Custom method to find our title
        return this._browser.text("h1.header-title")[0];
    }

    findKoalas() {
        return this._browser.findByTextContaining(/koala/);
    }

    _beforeOpen(options) { // This hook will be called anytime `browser.open` is executed
        // You can perform actions required for your plugin to run whenever
        // a new page is opened such as setting up cache
        // keep in mind that the page won't be accesible yet
    }

    _beforeClose() { // This hook will be called anytime `browser.close` is executed
        // You can perform actions required for your plugin when the page is
        // close, keep in mind that this will only be called on browser.close and
        // not on any page loading
    }

    _afterOpen(options) { // This hook will be called after the page is opened and loaded
        // You can use this hook to start performing actions with evaluate or
        // adding custom scripts with this._browser.page.addScriptTag
    }
}


class MyPluginAssertions { // The assertions will be under browser.assertions[myPluginName]
    constructor(browser, myPlugin) { // Plugin assertions receive browser and plugin in the constructor
        this._myPlugin = myPlugin;
    }

    thereAreKoalas(count) {
        const koalas = this._myPlugin.findKoalas().length;
        if (!count && koalas === 0) throw new AssertionError("No koalas :("); // node's AssertionError
        else if (count && koalas !== count) throw new AssertionError("No enough koalas :/");
    }

    headerTitle(title) {
        if (this._myPlugin.getHeaderTitle() !== title) throw new AssertionError("Invalid title");
    }
}

Wendigo.registerPlugin("koalafied", MyPlugin, MyPluginAssertions);

const browser=Wendigo.createBrowser();
//... more code ...

browser.koalafied.getHeaderTitle(); // Koalas are great

browser.assert.koalafied.headerTitle("Koalas are great");
browser.assert.koalafied.thereAreKoalas();
```

**Wendigo.registerPlugin** receives 3 parameters:   
* **name**: Name to be used to access the plugin, it must be different than other plugins and should not collide with wendigo core modules.
* **plugin**: Class to be used as plugin accessed under `browser.name`, this class will receive `browser` as constructor parameter and 2 methods can be implemented as hooks:
  * **_beforeOpen**: Called when `browser.open` is called, before opening the page.
  * **_beforeClose**: Called when `browser.close` is called, before closing the page.
* **assertion**: Class to be used as plugin's assertions, it can be accessed on `browser.assertion.name` the constructor will received both the browser and the core plugin as parameters

registerPlugin also accepts a single object containing the data in the following structure:
* `name`
* `plugin`
* `assertion`

Keep in mind that both the plugin and the assertions are optional, but at least one must exists to register the plugin.

Instead of classes, if a plain function is provided as a plugin or assertion, it will be attached directly to browser or browser assertion (without calling `new`), the function will receive the same arguments as the constructor of the plugin, as well as any extra parameter passed to the function:

```js
function myPluginAssertionFunc(browser, myPlugin, count){
    const koalas = myPlugin.findKoalas().length;
    if (!count && koalas === 0) throw new AssertionError("No koalas :("); // node's AssertionError
    else if (count && koalas !== count) throw new AssertionError("No enough koalas :/");
}

Wendigo.registerPlugin("koalafied", MyPlugin, MyPluginAssertions);
browser.assert.koalafied(); // note the assertion is called directly
```

## Publishing a plugin
If you want to create a new plugin and publish it in the npm store. Please, follow the following steps:

1. Make sure your package exports a single object compatible with the interface described above to make it easier to import. Do not export the classes individually unless you want them to be imported that way.
2. Make sure your code is tested using node 8 and above. Avoid using async/await if possible.
3. Set Wendigo as a [peer dependency](https://docs.npmjs.com/files/package.json#peerdependencies) in you package.json.
    * If you are writing tests, also set Wendigo as a dev dependency, **never** as a normal dependency.
4. Wendigo usually follows [semantic versioning](https://semver.org/) so your plugin should be compatible with any minor version above the version you wrote it, but a lot of things may break, so it is good to make sure your plugin still works properly in the latest version after a release.
5. Let people (and koalas!) know about it.

# Examples

**Testing a simple page with Mocha and Wendigo**

```javascript
const assert = require('assert');
const Wendigo = require('wendigo');

describe("My Tests", function() {
    this.timeout(5000); // Recommended for CI
    let browser;

    beforeEach(async() => {
        browser = await Wendigo.createBrowser();
    });

    afterEach(async() => {
        // For more speed, this method could be only executed after all tests pass
        // if your tests do not rely on state changes
        await browser.close();
    });

    after(async() => {
        await Wendigo.stop(); // After all tests finished
    });

    it("Page Title", async() => {
        await browser.open("http://localhost");
        await browser.assert.text("h1#main-title", "My Webpage");
        await browser.assert.title("My Webpage");
    });

    it("Open Menu", async() => {
        await browser.open("http://localhost");
        await browser.assert.not.visible(".menu");
        await browser.click(".btn.open-menu");
        await browser.assert.visible(".menu");
    });
});
```

# Development
These instructions assume node>8.0.0 and npm installed:

1. Clone the git repository (`dev` branch)
2. `npm install`
3. `npm test` to execute the tests
  * `npm run lint` to execute the linting tests
  * `npm run dummy-server` to start the testing server on port 8002

Before doing a commit or PR to the `dev` branch, make sure both the tests and lint tests pass.

## Architecture

* `Wendigo`: The main class exported by the module, provides the base interface to instantiate the browser class.
  * `BrowserFactory`: class takes care of the creation of the browser instance
* `Browser`: Provides all the API to connect, retrieve information and perform assertions on a webpage. The BrowserFactory will compose this class from multiple mixins and modules.
  * `mixins`: A Browser is divided in several mixins (found on the folder with the same name). These will be composed in the Browser class at runtime.
* `Modules`: A module (or plugin) represents an object that will be attached to the browser at runtime with a given name (e.g. localStorage).
  * Modules are different from mixins in that the modules are attached as a separate class instance whereas mixins are composed into the same class.
  * Note that each module is composed by 2 parts, one attached to the main browser class and other to the assertion module.
* `Injection Scripts`: These scripts will be injected at runtime into the browser, and are required to perform most of wendigo actions within the browser context, these scripts cannot access other parts of the code, and the rest of the code should not use these scripts out of the browser's context. These scripts can be used in the context of `browser.evaluate` by the user and other plugins
  * `ẀendigoUtils`: Several utilities required by Wendigo in the browser's context.
  * `WendigoQuery`: Wrapper of the different types of selectors (css, xpath and DOM node)

# Troubleshooting

## Error: Failed to launch chrome! No usable sandbox!
This error may appear when running wendigo on certain systems and in most CI services. The sandbox setup can be bypassed by setting the environment variable `NO_SANDBOX=true`.

For example `NO_SANDBOX=true npm test`.

## MaxListenersExceededWarning: Possible EventEmitter memory leak detected warning
This can be caused by executing `Wendigo.createBrowser` multiple times without calling `browser.close` on previous browsers, causing all of them to be kept open. This may cause performance issues in your tests and it is recommended to close **every** browser after using it.

## Running Tests With Travis CI
Running tests using Puppeteer's require disabling the sandbox running mode. This can easily be achieved by passing the environment variable `NO_SANDBOX=true`, this can be done either as part of the test execution command, as a Travis secret env variable or in the `.travis.yml` file itself. It is recommended to add `travis_retry` to allow travis to execute the tests multiple times, as browser-based setup may fail frequently on travis workers:

```yml
language: node_js
os:
    - linux
node_js:
    - "stable"
sudo: false
env:
  - NO_SANDBOX=true

script:
    - travis_retry npm test

cache:
  directories:
    - node_modules
```
_Example of travis.yml file_

### Running Tests With Gitlab CI

Using gitlab with the default node image requires installing a few dependencies with `apt-get` before installing wendigo. Same as in travis, sandbox mode should be disabled with the env variable `NO_SANDBOX`. It is recommended to add `retry: 2` to allow the CI to execute the tests multiple times, as browser-based setup may fail frequently on CI workers:

```yml
image: node:8.9.4

variables:
  NO_SANDBOX: "true"

before_script:
    - apt-get update
    - apt-get install -y -q gconf-service libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release xdg-utils wget
    - npm install

test:
  stage: test
  retry: 2
  script:
    - npm test
```
_Example of .gitlab-ci.yml_

### Assertion failed messages without error
If you are using node@10 and puppeteer 1.4.0 or less, you may experience messages such as `Assertion failed: No node found for selector`, this is due to a change in how `console.assertion` works in node 10 and how puppeteer uses it, these messages won't affect the tests, if the messages are a big problem for you, consider downgrading your node.js version, upgrading puppeteer if possible or overriding console.assert: `console.assert=()=>{}`.

> Remember to check [Puppeteer Troubleshooting](https://github.com/GoogleChrome/puppeteer/blob/master/docs/troubleshooting.md)

# Acknowledgements

* [Puppeteer](https://github.com/GoogleChrome/puppeteer) and Chrome Headless as base headless browser.
* [ZombieJs](https://github.com/assaf/zombie) as inspiration of the assertion library.
* [NightmareJs](http://www.nightmarejs.org) as inspiration for part of the browser interface.


# License

* Wendigo is maintained by @angrykoala under GPL-3.0 License
* Wendigo Logo, made by @jbeguna04 is licensed under [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/)
