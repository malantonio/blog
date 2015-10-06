---
title: Whisper down Electron Alley
layout: post
tagged: javascript, electron
---

[Electron menus][electron-menu] are kind of a drag to put together, especially
on OS X, where you _need_ to make sure you have a first-place app menu for
any of the others to show up. [Building them from a template][menu-template]
simplifies the work a lot, but takes up a ton of room. A way to get around
a ton of lines of menu-related code in your app's main .js file (let's call it
`app.js`) is to extract it into its own file and treat it as a module:

```javascript
// menu-template.js
module.exports = [
  {
    label: 'Kewl App',
    submenu: [
    // ... //
    ]
  }
]
```

and then calling it from within the main file:

```javascript
// app.js
var Menu = require('menu')
var template = require(__dirname + '/path/to/menu-template.js')
// ... //

app.on('ready', function () {
  var mainMenu = Menu.buildFromTemplate(template)
  Menu.setApplicationMenu(mainMenu)
  // yadda yadda
})
```

Which is all well and good until you want to call a function declared within
`app.js` which is outside of the scope of `menu-template.js` since the latter
is now a module. Now what? By my count, you can do four things:

1. Give up and only use your menu for boring things
2. Put your `menu-template.js` code back into `app.js` and have a ~100 line 
block of code that you just scroll past 99% of the time (hey, that's why 
code-folding is a thing amirite??)
3. Have `menu-template.js` instead export a function that takes the external
functions as parameters, either as themselves or as an object.
4. Use Electron's [`ipc`][electron-ipc] to communicate to the app via the 
focused window (spoiler, this is the one I'm gonna talk about)

> **Note:** if your app's got multiple windows running, this approach probably 
> isn't the best. I'd go with #3 and use an `opts` object instead.

Since the menu is constructed on the main process, its version of `ipc` can 
only listen for messages and isn't given an `ipc.send` function. However, when
building the `menu-item`, the `click` property is expected to be a callback
with two parameters: `item` and `focusedWindow`. We can use the `focusedWindow`
as a relay between the menu and the main process to pass along a message. So
something like this to open a config menu:

first:

```javascript
// menu-template.js
// ...
{
  label: 'Open Config Menu',
  click: function (item, focusedWindow) {
    // tell the focused window to tell the app to open the config
    focusedWindow.send('menu:open-config')
  }  
}
//...
```

then:

```javascript
// mainWindow.js
ipc.on('menu:open-config', function () {
  // hey app, the menu wants you to open the config!
  ipc.send('window:open-config')
})
```

finally:

```javascript
// app.js
ipc.on('window:open-config', function () {
  // okay!
  // opening the config window
})
```

> **Semi-pro tip:** namespace your `ipc` messages so you can keep track of what's
> coming from where.

And voila you're chattin' with the main process. This can also streamline `app.js` 
by only occupying one listener to open the config. So if you want to also put a
button on the `mainWindow`'s GUI that toggles the config window, you would only
need to add a listener to the button that sends the same `window:open-config`
message.

```javascript
// mainWindow.js
document.querySelector('.config-btn').addEventListener('click', function (ev) {
  ipc.send('window:open-config')
  return false
})
```

[electron-menu]: http://electron.atom.io/docs/latest/api/menu/
[menu-template]: http://electron.atom.io/docs/latest/api/menu/#menu-buildfromtemplate-template
[electron-ipc]: http://electron.atom.io/docs/latest/api/ipc-renderer/