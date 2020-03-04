![ForgeVuer](assets/images/forgeVuerLogo.png)
# ForgeVuer <!-- omit in toc -->

A Vue.js component providing an easy to setup, almost *"plug and play"* experience for Autodesk's Forge Viewer on Vue.js

- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installing](#installing)
- [TL;DR](#tldr)
- [Setup](#setup)
  - [Access Token](#access-token)
- [Properties](#properties)
- [Events](#events)
  - [Forge Viewer Events](#forge-viewer-events)
  - [ForgeVuer Events](#forgevuer-events)
- [Custom Extensions](#custom-extensions)
- [Versioning](#versioning)
- [License](#license)

## Getting Started

These instructions will get you started on how to install, use and customize the **ForgeVuer** component.

### Prerequisites

- A minimal **Vue** app to use the component.
- The latest Autodesk Forge Viewer **version 7** styling and javascript files referenced on the html `head` section.

```html
<head>
    [...]
    <!-- Autodesk Forge Viewer files -->
    <link rel="stylesheet" href="https://developer.api.autodesk.com/modelderivative/v2/viewers/7.*/style.min.css" type="text/css">
    <script src="https://developer.api.autodesk.com/modelderivative/v2/viewers/7.*/viewer3D.min.js"></script>
    [...]
</head>
```

### Installing
```
npm install forge-vuer
```

## TL;DR

A minimal working setup on a SPA application:

```html
<!-- App.vue-->
<template>
  <div id="app">
    <forge-vuer
      :getAccessToken="myGetTokenMethodAsync"
      :urn="myObjectUrn"
    />
  </div>
</template>

<script>
import ForgeVuer from 'forge-vuer';

export default {
    name: 'app',
    components: {
        ForgeVuer
    },
    data: () => {
        return {
            myToken:"{A VALID, NON_EXPIRED TOKEN CAN BE USE FOR TESTING PURPOSES}",
            myObjectUrn: "dXJuOmFkc2sub2JqZWN0czpvcy5vYmplY3Q6c2Rhc2Rhc2QvYnVubnkub2Jq",
        }
    },
    methods: {
        myGetTokenMethodAsync: async function(onSuccess){
            // An API call to retrieve a valid token should be
            // done here. A backend service might need to be implemented.

            // For testing purposes, a valid token can be hardcoded but will 
            // last a maximum of 1 hour (3600 seconds.)
            let token = this.myToken;
            let expireTimeSeconds = 3599;
            onSuccess(token, expireTimeSeconds);
        },
    }
}
</script>
```

## Setup

 Nevertheless, it requires some level of setup in order to have a secure and stable experience.

### Access Token

Forge Viewer requires to be associated to a valid Forge Application, and this is achieved by the use of an **access token** retrieved using the application's **CLIENT_SECRET** and **CLIENT_ID** credentials.

These credentials **MUST NOT** be exposed on the front-end as:
- Entails a security risk for your Forge Application.
- Making calls to Forge API from the front-end will likely return a **Cross Origin** error.

Instead, a backend service should be implemented so it securely returns a valid token and expiring time. An example of an endpoint for this purpose using **Express.js** and **Axios**:

```javascript
// Backend API
let app = new Express();

app.use("/api/token", async (req, res, next) => {
    return axios.post(
            "https://developer.api.autodesk.com/authentication/v1/authenticate",
            {
                client_id: "YOUR CLIENT_ID",
                client_secret: "YOUR CLIENT_SECRET",
                grant_type: "client_credentials&",
                scopes: "data:read"
            });
});
```
```html
<!-- SPA -->
<template>

    <forge-vuer
      [...]

      @getAccessToken="handleAccessToken"
    />
</template>

<script>

export default{
    [...]

    methods: {
        handleAccessToken: async function(onSuccess){
            axios.get(`/api/token`)
            .then(response => {
                onSuccess(response.data.access_token, response.data.expires_in);
            })
            .catch(error => {
                console.log(error);
            });
        }
    }

    [...]
}

</script>
```


## Properties
The component has some properties to configure and customize it.

| Prop             | Type       | Default      | Required | Description                                                                                                                                                                                                                                                                                                   |
| ---------------- | ---------- | ------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`             | `String`   | `forge-vuer` | `false`  | This defines the `id` attribute of the DOM element that will host the Viewer                                                                                                                                                                                                                                  |
| `getAccessToken` | `Function` | -            | `true`   | Function that will provide a valid access token to the Viewer by calling the `onSuccess` callback.                                                                                                                                                                                                            |
| `urn`            | `String`   | -            | `false`  | Urn of the file to load. Make sure the file has already been [translated](https://forge.autodesk.com/en/docs/model-derivative/v2/tutorials/prepare-file-for-viewer/).                                                                                                                                         |
| `options`        | `Object`   | -            | `false`  | Options used to [initialize](https://forge.autodesk.com/en/docs/viewer/v6/reference/Viewing/Initializer/#new-initializer-options-callback) the Viewer instance. The only property that will not be used is `getAccessToken`, as it is replace by the corresponding function passed as a component's property. |
| `headless`       | `Boolean`  | `false`      | `false`  | This property defines if the viewer is meant to be use in [headless](https://forge.autodesk.com/en/docs/viewer/v6/tutorials/headless/) mode.                                                                                                                                                                  |
| `extensions`     | `Object`   | -            | `false`  | Object containing the custom extensions. See [custom extensions](#custom-extensions) for more detail.                                                                                                                                                                                                         |



## Events
The component exposes two types of events to which can be subscribed: original Forge Viewer Events and Custom Events.

### Forge Viewer Events

As described on Forge Viewer [API documentation](https://autodeskviewer.com/viewers/latest/docs/Autodesk.Viewing.html#events), the viewer provides several events like `SELECTION_CHANGED_EVENT`, `PROGRESS_UPDATE_EVENT`, etc. The component allows to seamlessly subscribe to these event using the familiar vue syntax `v-on:` or `@` by the convention:
- Same name of original event but all lower cased.
- Underscores `_` replaced by hyphens/dashes `-`.

As an example:

| Original Event            | Subscribed on component   |
| ------------------------- | ------------------------- |
| `SELECTION_CHANGED_EVENT` | `@selection-change-event` |
| `PROGRESS_UPDATE_EVENT`   | `@progress-update-event`  |

Internally, on creation it will try to map the component's events to the corresponding on the Viewer, providing an easy interface to subscribe to any original event.
Any data associated that an event might return is encapsulated on a single object to allow for an automated mapping. This means that your subscribing function will have a single input argument containing all parameters passed by the event.

```html
<!-- SPA -->
<template>

    <forge-vuer
      [...]

      @progress-update-event="handleProgressUpdated"
    />
</template>

<script>

export default{
    [...]

    methods: {
        handleProgressUpdated: function(eventData){
            console.log(`Progress: ${eventData.percentage}%`)
        }
    }

    [...]
}

</script>
```


### ForgeVuer Events
Additionally, the component provides some additional events that allows to act when certain actions happen during the component's execution.

| Name                | Arguments           | Description                                                                                                                                                                                                                                        |
| ------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `error`             | `Error`             | This event is fired whenever any error that hasn't been handle in any other way (internally by Forge emitting their events or some other of these custom events). When fired, this event will have as input the actual error that has been thrown. |
| `documentLoading`   | -                   | Event fired when a new `urn` has been provided and the process of loading its associated document starts.                                                                                                                                          |
| `documentLoadError` | `Error`             | Fired when Forge fails to load a document. If no function is subscribed to this event, the default `Error` event will be thrown. The `Error` passed as argument contains the Forge `errorCode` reference.*                                         |
| `viewerStarted`     | `Viewer3D` instance | Event fired when the Viewer3D has been initialized, passing this instance as function argument.                                                                                                                                                    |
| `modelLoading`      | -                   | Fired when the model associated with the document starts to load.                                                                                                                                                                                  |
| `modelLoaded`       | `model`             | Fired when the model is successfully loaded. The argument is a [Model](https://forge.autodesk.com/en/docs/viewer/v6/reference/Viewing/Model/) instance.                                                                                            |
| `modelLoadError`    | `Error`             | Fired when Forge fails to load a model. If no function is subscribed to this event, the default `Error` event will be thrown. The `Error` passed as argument contains the Forge `errorCode` reference.*                                            |

> *For a detailed list of Forge ErrorCodes and their meaning, visit [this blog post](https://forge.autodesk.com/cloud_and_mobile/2016/01/error-codes-in-view-and-data-api.html)

## Custom Extensions (6.*)

One of the most powerful features of Autodesk Forge Viewer is the ability to add custom functionality via **Extensions**. Registering custom extensions to the component's Viewer instance can be done just through the `extensions` component property. The only difference with the common [examples](https://forge.autodesk.com/en/docs/viewer/v6/tutorials/extensions/) found is that the extension implementation must be wrapped within a function so the component can register them at runtime.

This would be a simple example of custom extension:
```js
// Example from https://forge.autodesk.com/en/docs/viewer/v6/tutorials/extensions/#step-2-write-the-extension-code
// my-awesome-extension.js

export default function (AutodeskViewing) {

    function MyAwesomeExtension(viewer, options) {
        AutodeskViewing.Extension.call(this, viewer, options);
    }

    MyAwesomeExtension.prototype = Object.create(AutodeskViewing.Extension.prototype);
    MyAwesomeExtension.prototype.constructor = MyAwesomeExtension;

    MyAwesomeExtension.prototype.load = function () {
        alert('MyAwesomeExtension is loaded!');
        return true;
    };

    MyAwesomeExtension.prototype.unload = function () {
        alert('MyAwesomeExtension is now unloaded!');
        return true;
    };

    // Is not necessary to implicitly register the extension
    // as this is handled by the component.
    // Autodesk.Viewing.theExtensionManager.registerExtension('MyAwesomeExtension', MyAwesomeExtension);

    // IMPORTANT to return the extension function itself.
    return MyAwesomeExtension;

}
```

If you are comfortable and able to use ES6 classes, an extension could also be written as:
```js
// Example from https://forge.autodesk.com/en/docs/viewer/v6/tutorials/toolbar-button/#step-1-detect-the-toolbar
// my-custom-toolbar.js

export default function (AutodeskViewing) {

    return class ToolbarExtension extends AutodeskViewing.Extension {
        viewer;
        options;
        subToolbar;

        constructor(viewer, options) {
            super(viewer, options);
            this.viewer = viewer;
            this.options = options;
        }
        
        load = function () {
            
            if (this.viewer.toolbar) {
                // Toolbar is already available, create the UI
                this.createUI();
            } else {
                // Toolbar hasn't been created yet, wait until we get notification of its creation
                this.onToolbarCreatedBinded = this.onToolbarCreated.bind(this);
                this.viewer.addEventListener(AutodeskViewing.TOOLBAR_CREATED_EVENT, this.onToolbarCreatedBinded);
            }

            return true;
        }

        unload = function () {
            this.viewer.toolbar.removeControl(this.subToolbar);
            return true;
        };

        onToolbarCreated = function() {
            this.viewer.removeEventListener(AutodeskViewing.TOOLBAR_CREATED_EVENT, this.onToolbarCreatedBinded);
            this.onToolbarCreatedBinded = null;
            this.createUI();
        };

        createUI = function() {
            var viewer = this.viewer;

            // Button 1
            var button1 = new AutodeskViewing.UI.Button('my-view-front-button');
            button1.onClick = function() {
                viewer.setViewCube('front');
            };
            button1.addClass('my-view-front-button');
            button1.setToolTip('View front');

            // Button 2
            var button2 = new AutodeskViewing.UI.Button('my-view-back-button');
            button2.onClick = function() {
                viewer.setViewCube('back');
            };
            button2.addClass('my-view-back-button');
            button2.setToolTip('View Back');

            // SubToolbar
            this.subToolbar = new AutodeskViewing.UI.ControlGroup('my-custom-view-toolbar');
            this.subToolbar.addControl(button1);
            this.subToolbar.addControl(button2);

            viewer.toolbar.addControl(this.subToolbar);
        };
    }

}
```

Then, on the component implementation we just need to set the `extension` property as an object where `keys` will be the registered `id` of the extensions, and the values the imported functions.

```html
<!-- SPA -->
<template>
    <forge-vuer
      [...]

        extensions="{
            'myAwesomeExtension': myAwesomeExtension,
            'myCustomToolbar': myCustomToolbar,
        }"
    />
</template>

<script>
import myAwesomeExtension from './path/to/my-awesome-extension.js';
import myCustomToolbar from './path/to/my-custom-toolbar.js';

export default{
    [...]
}

</script>
```

## Custom Extensions (7.*)

There has been [breaking changes](https://forge.autodesk.com/blog/breaking-change-forge-viewerloadextension) since 6.*. Also, you need [CSS classes](https://forge.autodesk.com/blog/what-icons-are-provided-viewer-stylesheet) to set the icon of the buttons.

```js
// Example from https://forge.autodesk.com/en/docs/viewer/v6/tutorials/toolbar-button/#step-1-detect-the-toolbar
// my-custom-toolbar.js

export default function (AutodeskViewing) {

    return class ToolbarExtension extends AutodeskViewing.Extension {
        viewer;
        options;
        subToolbar;

        constructor(viewer, options) {
            super(viewer, options);
            this.viewer = viewer;
            this.options = options;
        }

        load = function () {
            if (this.viewer.toolbar) {
                // Toolbar is already available, create the UI
                this.createUI();
            } else {
                // Toolbar hasn't been created yet, wait until we get notification of its creation
                this.onToolbarCreatedBinded = this.onToolbarCreated.bind(this);
                this.viewer.addEventListener(AutodeskViewing.TOOLBAR_CREATED_EVENT, this.onToolbarCreatedBinded);
            }

            return true;
        }

        unload = function () {
            this.viewer.toolbar.removeControl(this.subToolbar);
            return true;
        };

        onToolbarCreated = function () {
            this.viewer.removeEventListener(AutodeskViewing.TOOLBAR_CREATED_EVENT, this.onToolbarCreatedBinded);
            this.onToolbarCreatedBinded = null;
            this.createUI();
        };

        createUI = async function () {
            let viewer = this.viewer;
            const vC = await viewer.loadExtension('Autodesk.ViewCubeUi');

            // Button 1
            let button1 = new AutodeskViewing.UI.Button('my-view-front-button');
            button1.onClick = function () {
                vC.setViewCube('front');
            };
            button1.addClass('my-view-front-button');
            button1.setToolTip('View front');
            button1.setIcon('adsk-icon-first'); 

            // Button 2
            let button2 = new AutodeskViewing.UI.Button('my-view-back-button');
            button2.onClick = function () {
                vC.setViewCube('back');
            };
            button2.addClass('my-view-back-button');
            button2.setToolTip('View Back');
            button2.setIcon('adsk-icon-second');

            // SubToolbar
            this.subToolbar = new AutodeskViewing.UI.ControlGroup('my-custom-view-toolbar');
            this.subToolbar.addControl(button1);
            this.subToolbar.addControl(button2);

            viewer.toolbar.addControl(this.subToolbar);
        };
    }
}
```

Moreover, the function for checking load module has been changed (See `/services/Utils.js`):
```js
        // If extension already registered
        if (AutodeskViewing.theExtensionManager.isAvailable(name)) {
            registeredExtensions.push(name);
            continue;
        }
```

## Detailed implementation note (nuxt.js, 2 legged authtication, OSS bucket approch)

Note that Hub approch requires 3 legged oauth. If you are not going to implement redirection and user management, you can use OSS bucket approch instead.
Follow session **Access Token** and perform a online 2 legged or 3 legged authentication.
If you are using Hub approch, follow [this guide](https://github.com/Autodesk-Forge/forge-derivatives-explorer) or upload files [here](https://derivatives.autodesk.io/), [a360](https://a360.autodesk.com/drive/app/) or [another viewer](https://viewer.autodesk.com/)
If you are using Bucket approch, follow [this guide](https://forge.autodesk.com/en/docs/data/v2/tutorials/app-managed-bucket/). If you are feeling puzzled, read [this article](https://forge.autodesk.com/blog/oss-manager-migrated-autodeskio-server) and manually upload the files [in their new website](https://oss-manager.autodesk.io/).

If you are using plain Vue, `/sample/index.html` will be enough for you. Host it on a web host, host domain must match with the app in forge web console.
If you are using SSR framework (e.g. [Nuxt](https://nuxtjs.org/) ), you are recommended to directly copy the source codes in `/src` into `/components`, even with inner folder (i.e. `/components/forge`). `/src/index.js` can be omited, as nuxt will handle it instead.
Then wrap it into a plugin (e.g. `/plugins/forge-vuer.js`):

```js
import Vue from 'vue';
//import ForgeVuer from 'forge-vuer';
import ForgeVuer from '~/components/forge/ForgeVuer'

Vue.component('forge-vuer', ForgeVuer);
```

Since you have no direct access in head session, you can use `head()` to tell nuxt to add them. ([Article](https://nuxtjs.org/api/pages-head/)) 
Adding them into vue page will minimalize the effect. Extensions can be a k-v map e.g. `/pages/forge.vue`:
```vue
<template> 
    <no-ssr placeholder="Loading...">
        <forge-vuer
          :getAccessToken="handleAccessToken"
          :urn="myObjectUrn"
          @progress-update-event="updateProgress"
          @selection-changed-event="spamLog"
          @viewerStarted="spamLog"
          @error="errorLog"
          :style="mapStyle"
          :extensions="extensions"
        />
    </no-ssr>
</template>
<scripts>

<script>
import myAwesomeExtension from "~/components/forge/extensions/myAwesomeExtension";
import myCustomToolbar from "~/components/forge/extensions/myCustomToolbar";
...
export default {
    head() {
        return {
        script: [
            {
            src:
                "https://developer.api.autodesk.com/modelderivative/v2/viewers/7.*/viewer3D.min.js"
            } //defer: true
        ],
        link: [
            {
            rel: "stylesheet",
            href:
                "https://developer.api.autodesk.com/modelderivative/v2/viewers/7.*/style.min.css"
            }
        ]
        };
    },
    data() {
        return {
            tokenPkg: {},
            treeNodePkg: {},
            modelProgress: 0,
            extensions: { 
                myAwesomeExtension, 
                myCustomToolbar 
            }
        };
    },
  ...
</scripts>
```

Finally tell nuxt to load the plugin with SSR disabled (`nuxt.config.js`):
```js
  /*
  ** Plugins to load before mounting the App
  */
  plugins: [
    //'@/plugins/vuetify',
    //...
    { src: '@/plugins/forge-vuer', ssr: false },
  ],
```

## Versioning

We use [SemVer](http://semver.org/) for versioning.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
