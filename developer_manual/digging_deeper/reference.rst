=========
Reference providers
=========

Reference providers are related with two Nextcloud features:

* Link previews
* The link picker

Link previews were introduced in Nextcloud 25. are rendered in...

The link picker...

App developers can register their own reference providers to:

* add support for new kinds of HTTP links by:
    * resolving the links, getting information on the link targets
    * optionally provide their own reference widgets
* extend the link picker with:
    * supporting existing unified search providers
    * optionally register custom picker components


Register a reference provider
---------------------------

A reference provider is represented by a class implementing the `OCP\\Collaboration\\Reference\\IReferenceProvider`


Resolving links
---------------------------

Using the default widget
^^^^^^^^^^^^^^^^^^^^^^^^^^^

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
