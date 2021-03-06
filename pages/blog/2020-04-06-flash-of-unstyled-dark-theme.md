# FOUDT: Flash of Unstyled Dark Theme

---

<Intro>
As many of us are embarking on the dark theme website trend, the popularity of building statically generated websites with frameworks like Gatsby and Next is also growing. This has given rise to a new unintentional side effect — <em>the flash of unstyled dark theme</em>. Learn what it is, and how to avoid it.
</Intro>

!["DARK THEME"](/gatsby-dark-theme-lighthouse.jpg "Running a lighthouse test again gatsbyjs.org (as of 2020-04-02) reveals that the page has pre-rendered a light theme and then re-renders with a dark theme once client-side JavaScript kicks in")


## Naive Implementation

Statically generated sites are "pre-rendered" HTML files usually served from CDN. As part of pre-rendering CSS is also applied, usually with some CSS-in-JS framework involved. On first paint, the user will see a pre-rendered page. Once client-side JavaScript runs and detects the users preferred theme, the page will re-render.

If the users preferred theme is different from the default theme, the page re-render will be perceived as a "flash" between the two. I call this the *naive theming implementation*.


### Is this *really* a problem?

From a UX perspective, this is similar to other undesirable side effects such as [flash of unstyled content](https://en.wikipedia.org/wiki/Flash_of_unstyled_content) and [flash of faux text](https://www.zachleat.com/web-fonts/demos/foft.html). It might not be the end of the world but it's still noticeable. On a slow connection it can be very distracting and confusing as it takes a longer time for client-side JavaScript to execute.

## Improved implementation

It turns out this side-effect is entirely avoidable with some basic usage of CSS media queries and global [CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties) (commonly referred to as CSS variables)

If you're just rendering the page based on user OS theme preference, this problem can **and should** be solved in CSS, not in JavaScript. Use the `prefers-color-scheme` media query in the `<head>` section of your HTML.

```css
:root {
  --color-fg: #000;
  --color-bg: #fff;
}
@media (prefers-color-scheme: dark) {
  :root {
    --color-fg: #fff;
    --color-bg: #000;
  }
}
```

In your React components simply refer to your global custom properties.

```jsx
// Pseudo code
const Text = ({ children }) {
  <P css={{ color: "var(--color-fg" }}>
    {children}
  </P>
}
```

Or if you're using good old global CSS, just do something like:

```jsx
body {
  color: var(--color-fg);
  background: var(--color-bg);
}
```

As we've now moved the theming logic to CSS, there should be no flash as client-side rendering kicks in.

One drawback is that CSS custom properties has [limited support](https://caniuse.com/#search=custom%20properties) in legacy browsers (notably Internet Explorer), but it can be [polyfilled](https://jhildenbiddle.github.io/css-vars-ponyfill/). I would find it acceptable to have a fallback for those browsers to only supported a single theme.


## Adding more features

The global CSS custom properties solution works well for the simple use case of rendering a single theme. What about the more complex use cases?

### Allow users to switch themes

A common feature is to allow users to switch between themes after the initial rendering. For this kind of interactivity we need JavaScript. That does not imply that we need React, we simply need an interface to modify the global CSS custom properties. Let's start with extracting our theme into a JSON structure:

```javascript
var themes = {
  light: {
    "color-fg": "#000",
    "color-bg": "#fff",
  },
  dark: {
    "color-fg": "#000",
    "color-bg": "#fff",
  }
};
```

Now you can add this piece of *render-blocking* JavaScript in your `<head>`

```javascript
// Get root <html> node style interface
var root = document.documentElement.style;
var darkMatcher = window.matchMedia("(prefers-color-scheme: dark)");
var prefersColorScheme = darkMatcher.matches ? "dark" : "light";

/*
 * You can overwrite this function from within your React app.
 * eg: window.__onSetTheme = (themeName) => setThemeName(themeName)
 */
window.__onSetTheme = function() {};
window.__setTheme = function(themeName) {
  var theme = themes[themeName];
  Object.keys(theme).forEach(function(key) {
    // Set global custom properties on root element
    root.setProperty("--" + key, theme[key]);
    window.__onSetTheme(theme)
  });
}
window.__setTheme(prefersColorScheme);
```

This will set the color scheme for the initial render and also expose a global `__setTheme` function that you can call from within your React UI. You can also override the `__onSetTheme` to subscribe to when a theme change is applied.


### Persist user theme preference

The next thing you might want to do is to persist the theme that the user has opted for. A good candidate for this is to use `localStorage`. Update the `__setTheme` function

```javascript
window.__setTheme(themeName) {
  window.localStorage.setItem("themeName", themeName);
  ...
}
```

Update the `setTheme` function call to attempt to use the persisted theme first, otherwise pass the `prefersColorScheme`

```javascript
var savedTheme = window.localStorage.getItem('themeName');
window.__setTheme(savedTheme || prefersColorScheme);
```

Ps. Consider wrapping your `localStorage` calls in a try/catch block statement to avoid any [runtime errors](https://stackoverflow.com/questions/18860098/on-a-browser-sessionstorage-in-safaris-private-browsing-does-not-work-the-same).

### Listen to thematic changes in user OS

As a last added bonus, we can listen to if the user changes the preferred OS color scheme. It can be debatable if the page should should change if the user has explicitly chosen a theme to use. I can see this being more useful for apps that always should adapt to it's surroundings like a desktop or mobile application.

```javascript
darkmatcher.addListener(function() {
  setTheme(darkMatcher.matches ? 'dark' : 'light');
})
```

Note: If you're using a pure CSS media query solution as outlined in the first improved example, you already get this for free without any additional JavaScript.


### Working examples

For the sake of this article I've put together two working demos to show how everything fits together. They include the React UI parts and local state management wired together with the theme switching JavaScript functions.

- Next example: [Demo](https://next-flash-of-unstyled-dark-theme.netlify.com/) | [Source](https://github.com/danielstocks/flash-of-unstyled-dark-theme/tree/master/next)
- Gatsby example: [Demo](https://gatsby-flash-of-unstyled-dark-theme.netlify.com/) | [Source](https://github.com/danielstocks/flash-of-unstyled-dark-theme/tree/master/gatsby)

## Summary

Hopefully you've now learned how to apply theming on a statically generated website built with React while avoiding the *flash of unstyled dark theme* side effect!

What we've covered in here:

- The naive theming approach
- The CSS only solution
- Adding Features
  - Theme switching
  - Persisting theme preference
  - Subscribing to OS level theme changes
- Next and Gatsby examples

A topic that we haven't covered in this article that I highly recommend studying further is that of [theming imagery](https://medium.com/dev-channel/re-colorization-for-dark-mode-19e2e17b584b).


### Credits

- Thanks to [Geries](https://www.gtothesquare.com/) & [Robin](https://weser.io) for reviewing this article

- The article title "Flash of Unstyled dark Theme" was taken from a [tweet](https://twitter.com/zachleat/status/1245453351950716936) by [Zach Leatherman](https://www.zachleat.com/)




