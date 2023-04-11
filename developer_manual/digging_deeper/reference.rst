=========
Reference providers
=========

Reference providers are related with two Nextcloud features:

* Link previews
* The smart picker

Link previews were introduced in Nextcloud 25.
To provide link previews, we need to:

* Resolve links (get information about links)
* Render links (show this information in the user interface)

The smart picker was introduced in Nextcloud 26. This is a user interface component
to allow users to search or generate links from various places in Nextcloud.

App developers can register their own reference providers to:

* add support for new kinds of HTTP links by:
    * resolving the links, getting information on the link targets
    * optionally provide their own reference widgets to have a custom preview rendering
* extend the link picker with:
    * supporting existing unified search providers
    * optionally register custom picker components

Display link previews
---------------------------

You need to make sure the `OCP\\Collaboration\\Reference\\RenderReferenceEvent` is dispatched
before you load the page where you want to display link previews or the link picker.

Link previews will be automatically rendered in the `NcRichText` component.

You can also render the preview of one specific link by using the `NcReferenceWidget` component.

Implement and register a reference provider
---------------------------

A reference provider is a class implementing the `OCP\\Collaboration\\Reference\\IReferenceProvider` interface.
If you just want to resolve links, implement the `IReferenceProvider` interface.

If you want your reference provider to be used by the smart picker, you need to extend the
`OCP\\Collaboration\\Reference\\ADiscoverableReferenceProvider` class to declare all required information.
There are 2 ways to appear in the smart picker.

* Either your reference provider implements the
`OCP\\Collaboration\\Reference\\ISearchableReferenceProvider` interface and you declare a list of unified search providers
that will be used in the smart picker
* or you don't implement this `ISearchableReferenceProvider` interface and make sure you register a custom picker component in the frontend.
This is described later in this documentation.

Resolving links
---------------------------

`NcRichText`, `NcReferenceList` and `NcReferenceWidget` will resolve and render links for you.
If you need to

Using the default widget
^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you are fine with the default

Using custom reference widgets
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Use the reference provider in the link picker
---------------------------

For you reference provider to appear in the link picker, it needs to be discoverable
(extend the `OCP\\Collaboration\\Reference\\ADiscoverableReferenceProvider` abstract class)
and either
* support one or multiple unified search providers
* register a custom picker component

This is an exclusive choice. You can't support search providers AND register a custom picker component.
If you still want to mix both, you can register a custom picker component which includes a search feature.

Extending `ADiscoverableReferenceProvider` implies defining those methods:

* `getId`: returns an ID which will be used by the link picker to identify this provider
* `getTitle`: returns a (ideally translated) provider title visible in the link picker provider list
* `getOrder`: returns an integer to help sorting the providers. The sort order is later superseeded by last usage timestamp
* `getIconUrl`: returns the URL of the provider icon, same as the title, the icon will be visible in the provider list

Declare supported unified search providers
^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you want your reference provider to let users pick links from unified search results, your reference provider must
implement `OCP\\Collaboration\\Reference\\ISearchableReferenceProvider` and define the `getSupportedSearchProviderIds`
method which return a list of supported search provider IDs.

Once this provider is selected in the link picker, users will see a generic search interface giving results from
all the search providers you declared as supported. Once a result is selected, the link picker will return
the associated resource URL.

Register a custom picker component
^^^^^^^^^^^^^^^^^^^^^^^^^^^

On the bakend side, in your `lib/AppInfo/Application.php`, you should listen to the
`OCP\\Collaboration\\Reference\\RenderReferenceEvent`. In the corresponding listener, you should load
the scripts that will register custom picker components.

You can implement your own picker interface by registering a custom picker component. This can be done with the
`registerCustomPickerElement` function from `@nextcloud/vue-richtext` (>= 2.1.0-beta.5).
This function takes 3 parameters:

* The reference provider ID for which you register the custom picker component
* The callback function to create and mount your component
* The callback function to delete/destroy your component

The creation callback must return a `CustomPickerRenderResult` object to which you have to give the DOM element
you just created and optionally an object (the Vue instance for example).
They will be then accessible in the destroy callback to let you properly clean and delete your custom component.

To register a Vue component:

.. code-block:: javascript

    import {
        registerCustomPickerElement,
        CustomPickerRenderResult,
    } from '@nextcloud/vue-richtext'
    import Vue from 'vue'
    import MyCustomPickerElement from './MyCustomPickerElement.vue'

    registerCustomPickerElement('YOUR_REFERENCE_PROVIDER_ID', (el, { providerId, accessible }) => {
        const Element = Vue.extend(MyCustomPickerElement)
        const vueElement = new Element({
            propsData: {
                providerId,
                accessible,
            },
        }).$mount(el)
        return new CustomPickerRenderResult(vueElement.$el, vueElement)
    }, (el, renderResult) => {
        renderResult.object.$destroy()
    })

To register anything else:

.. code-block:: javascript

    import {
        registerCustomPickerElement,
        CustomPickerRenderResult,
    } from '@nextcloud/vue-richtext'

    registerCustomPickerElement('YOUR_REFERENCE_PROVIDER_ID', (el, { providerId, accessible }) => {
        const paragraph = document.createElement('p')
        paragraph.textContent = 'click this button to return a hardcoded link'
        el.append(paragraph)
        const button = document.createElement('button')
        button.textContent = 'I am a button'
        button.addEventListener('click', () => {
            const event = new CustomEvent('submit', { bubbles: true, detail: 'https://nextcloud.com' })
            el.dispatchEvent(event)
        })
        el.append(button)
        return new CustomPickerRenderResult(el)
    }, (el, renderResult) => {
        renderResult.element.remove()
    })

In your custom component, just emit the `submit` event with the result as the event's data to pass it back to the link picker.
You can also emit the `cancel` event to abort and go back.
