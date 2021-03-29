# React AMP Design Pattern

## Summary

Build AMP with Web React components and extract only required CSS for a page:

- Hybrid AMP components allow developers to share UI code between web and AMP. Where AMP code is never included in web application bundles.
- With AMP First components, developer can create a separate AMP-only component for use cases where most of the component's code required for AMP is different from web.
- `Stylesheet` component extracts pagewise CSS.

## Basic Example

### Hybrid AMP component example

This example renders an Offers Banner with a Close button. When Close button is clicked, banner returns `null` for web and gets hidden for AMP.

```jsx
import { Stylesheet } from "reamp";
import * as bannerStyles from "./OfferBanner.scss";

function OfferBanner() {
  const [isVisible, setIsVisible] = useState(true);

  function handleCloseClick() {
    setIsVisible(false);
  }

  return isVisible ? (
    <>
      <Stylesheet id={bannerStyles._id_} css={bannerStyles._css_} />

      <div $amp-id="banner">
        <p>10 Offers Available!</p>
        <button onClick={handleCloseClick} $amp-on="tap: banner.hide">
          Close
        </button>
      </div>
    </>
  ) : null;
}
```

This example illustrates a few key points:

- With small modification a React component can be used for AMP
- AMP attributes use the `$amp-` prefix. Bundlers will treat these attributes as AMP-only and will not include them in web application bundles. They will only be included in AMP's server bundle.
- `Stylesheet` component accepting CSS Modules's style and id as props which is used in CSS extraction.

### AMP First component example

This example shows 2 Image components, one for web and another for AMP, rendering respective elements:

```jsx
// Image.amp.js
function Image(props) {
  return <amp-img {...props} />;
}
```

```jsx
// Image.js
function Image(props) {
  return <img {...props} />;
}
```

AMP-only components use the `.amp.js` suffix. Bundlers will resolve and include web application components into web application bundles and AMP-only components into AMP bundle.

## Motivation

A React component can be broken down into following parts:

- **Data**: Data to display on a screen
- **UI**: HTML and CSS code
- **Actions and Events**: User interaction code

Actions and Events code cannot be reused since AMP handles interactions with different API. But Data and UI can be reused to build AMP on Server.

Due to code reusability, web application and AMP UI will always be in-sync and reduces the effort of maintaining two different stacks for creating web pages and AMP.

## Detailed Design

### AMP attributes

AMP attributes use the `$amp-` prefix to differentiate it from other attributes. Bundlers with help of a babel plugin will include these attributes only in AMP bundles and remove from web application bundles.

### Babel plugin

A babel plugin will remove all AMP attributes with `$amp-` prefix using RegEx while bundling the components for web application.

Babel configuration in web:

```jsx
// web.babel.config.js
{
	"plugins": [
		[
			"reamp/babel-plugin",
			{
				"isAMP": false, // transpile code for web application
			}
		]
	],
}
```

Component before build:

```jsx
// Hybrid AMP component
<button onClick={handleCloseClick} $amp-on="tap: banner.hide">
  Close
</button>
```

Component after build:

```jsx
// Web bundle file
<button onClick={handleCloseClick}>Close</button>
```

Since all AMP attributes are removed during build, its code will never be included in web application bundles.

But for AMP, babel plugin will convert AMP attributes into valid syntax by removing `$amp-` prefix using RegEx.

Babel configuration in AMP:

```jsx
// amp.babel.config.js
{
	"plugins": [
		[
			"reamp/babel-plugin",
			{
				"isAMP": true, // transpile code for AMP application
			}
		]
	],
}
```

Component before build:

```jsx
// Hybrid AMP component
<button onClick={handleCloseClick} $amp-on="tap: banner.hide">
  Close
</button>
```

Component after build:

```jsx
// AMP bundle file
<button onClick={handleCloseClick} on="tap: banner.hide">
  Close
</button>
```

This babel plugin will handle Hybrid AMP components, but for AMP-only components webpack configuration will be required to resolve files based on their extensions so that AMP will import `*.amp.js` files and web applications will import `*.js` files.

webpack configuration in AMP:

```jsx
// amp.webpack.config.js
{
	resolve: {
		extensions: ['.amp.js', '.js'],
	}
}
```

webpack configuration in web:

```jsx
// web.webpack.config.js
{
	resolve: {
		extensions: ['.js'],
	}
}
```

Usage in React component:

```jsx
// App.js
import OfferBanner from "./OfferBanner"; // import without file extension
```

For AMP elements requiring any scripts, use [react-helmet](https://github.com/nfl/react-helmet) library:

```jsx
// AMP-only component
<>
  <Helmet>
    <script
      async
      custom-element="amp-bind"
      src="https://cdn.ampproject.org/v0/amp-bind-0.1.js"
    ></script>
  </Helmet>
  <p data-amp-bind-text="'Hello ' + foo">Hello World</p>
</>
```

And pass a custom `$amp-only` prop to `Helmet` component (can also be passed to any other React component) for Hybrid AMP component. Babel plugin will find this prop and remove the component from web application bundles:

```jsx
// Hybrid AMP component
<>
  <Helmet $amp-only>
    <script
      async
      custom-element="amp-bind"
      src="https://cdn.ampproject.org/v0/amp-bind-0.1.js"
    ></script>
  </Helmet>
  <p $amp-data-amp-bind-text="'Hello ' + foo">Hello World</p>
</>
```

### CSS Extraction

`Stylesheet` **component**

With `Stylesheet` component, CSS Modules style required for a page can be extracted on Server.

It maintains a map of component's style that were rendered for a requested page and Server will include this mapped style in head tag and return it.

As `Stylesheet` component is required only for AMP, babel plugin will remove it from web application bundles.

In a React component:

```jsx
// Hybrid AMP component
import { Stylesheet } from "reamp";
import * as bannerStyles from "./Banner.scss";

<>
  <Stylesheet id={bannerStyles._id_} css={bannerStyles._css_} />
  <button>Close</button>
</>;
```

On the server:

```jsx
// amp-server.js
const app = ReactDOM.renderToString(<App />);
const css = Stylesheet.getCSS();

<head>
	<style id="reamp-css">
		{css}
	</style>
</head>
<body>
	<div id="root">{app}</div>
</body>
```

**API**

**`css`**: string

Component's CSS code.

> Read "**CSS Extractor Plugin**" section to understand how CSS code will be made available to pass as prop value.

**`id`**: string

Hash id of component's CSS Modules file. This prop is required to avoid duplicate style from getting added in style map.

---

**Extracting CSS for Web**

`Stylesheet` component can also be used for web on the client-side to extract page specific CSS with below modifications:

- [server]: Along with returning CSS, server will also return a JSON with ids (i.e. rendered component's `id` prop values)

  On the server:

  ```jsx
  // web-server.js
  const app = ReactDOM.renderToString(<App />);
  const [css, ids] = Stylesheet.getCSS();

  <head>
  	<style id="reamp-css">
  		{css}
  	</style>
  </head>
  <body>
  	<div id="root">{app}</div>

  	<script id="__REAMP_CSS_IDS__" type="application/json">
  		{ids}
  	</script>
  </body>
  ```

  Response HTML page:

  ```html
  <head>
    <style id="reamp-css">
      ._svbcn {
        color: red;
      }
      ._jvbks {
        color: blue;
      }
      // ...
    </style>
  </head>
  <body>
    <script id="__REAMP_CSS_IDS__" type="application/json">
      ["1296312", "4763049", "4097534", "5986079", ...]
      // instead of returning an array,
      // we can return an object with key-value pairs for better performance
    </script>
  </body>
  ```

- [client]:
  - During initialisation `Stylesheet` component will initialise its style map object with JSON returned by Server.
  - As components start getting rendered, `Stylesheet` component will keep adding their ids in style map and append their CSS in DOM if not already present.

---

**CSS Extractor Plugin**

A CSS Extractor webpack plugin (similar to [mini-css-extract-plugin](https://github.com/webpack-contrib/mini-css-extract-plugin)) will provide `_id_` and `_css_` values while extracting CSS (generated from CSS Modules files) into separate files.

webpack configuration:

```jsx
// amp.webpack.config.js
{
  module: {
    rules: [
      {
        test: /\.scss$/i,
        use: [CSSExtractorPlugin.loader, "css-loader"],
      },
    ];
  }
}
```

This plugin can also be configured for web applications to extract CSS on client-side.

## Helper functions

### Conditional rendering

Use `IF` helper function for use cases were a component is to be conditionally rendered on web but is required to always render on AMP.

```jsx
import { AmpHelpers } from "reamp";

// Always render `AppBanner` component for AMP
AmpHelpers.IF(isVisible, true) && <AppBanner />;
```

```jsx
// AmpHelpers.js
function IF(webCondition, ampCondition) {
  if (typeof ampCondition !== "undefined") {
    return ampCondition;
  }

  return webCondition;
}
```

## Library

All the above discussed components, plugins and helper functions will be published in [`reamp`](https://github.com/dutiyesh/reamp) library by next week.

## Credits and Prior Art

React AMP synthesize ideas from several different sources:

- [react-helmet](https://github.com/nfl/react-helmet)'s implementation which internally uses [react-side-effect](https://github.com/gaearon/react-side-effect) library
- [Next.js AMP](https://nextjs.org/docs/api-reference/next/amp) support
- [react-amphtml](https://github.com/dfrankland/react-amphtml)

---

_Many thanks to [React RFCs](https://github.com/reactjs/rfcs) for the inspiration with this documentation format._
