=========
Reference providers
=========

Reference providers are related with two Nextcloud features:

* Link previews
* The Smart Picker

Link previews were introduced in Nextcloud 25.
Apps can register a reference provider to add preview support for new HTTP links.
To provide link previews, a provider needs to:

* Resolve the links (get information about the links)
* Render links (show this information in the user interface)

The Smart Picker was introduced in Nextcloud 26. This is a user interface component
allowing users to search or generate links from various places in Nextcloud like Text,
Talk, Collectives, Notes, Mail...
Reference providers can be implemented so they appear in the provider list of the Smart Picker.
The Smart Picker can show 2 types of providers:

* the ones to search (using existing Unified Search providers)
* the ones implementing their own custom picker component

In summary, reference providers can be registered by apps to

* add support for new kinds of HTTP links
    * resolve the links, get information on the link targets
    * optionally provide their own reference widgets to have a custom preview rendering
* extend the Smart Picker
    * use existing Unified Search providers
    * or optionally register custom picker components to have a specific user interface

Display link previews
---------------------------

If you just want to use the reference system to display link previews without extending it,
you need to make sure the ``OCP\Collaboration\Reference\RenderReferenceEvent`` is dispatched
before you load the page where you want to display link previews.

For example, you can place this before returning your TemplateResponse in your controller:

.. code-block:: php

    $this->eventDispatcher->dispatchTyped(new OCP\Collaboration\Reference\RenderReferenceEvent());

This is done in Text and Talk if you need more examples.

On the frontend side, there are 2 ways to display link previews (also named reference widgets):

NcRichText
~~~~~~~~~~~~~

Link previews will be automatically rendered for links in the content of the ``<NcRichText>`` Vue component.
This component will take care of resolving the links itself.

.. code-block:: html

    <NcRichText :text="message"
		:arguments="richParameters"
		:autolink="true"
		:reference-limit="0" />

NcRichText can be imported like this:

.. code-block:: javascript

    import NcRichText from '@nextcloud/vue/dist/Components/NcRichText.js'


`NcRichText doc <https://nextcloud-vue-components.netlify.app/#/Components/NcRichText?id=ncrichtext-1>`_

NcReferenceWidget
~~~~~~~~~~~~~

You can display a preview for a specific link by using the ``<NcReferenceWidget>`` component.
You need to ask the server to resolve the link to get a reference object that you can then give as a property
to NcReferenceWidget

To resolve a link:

.. code-block:: javascript

    const myLink = 'https://github.com'
    axios.get(generateOcsUrl('references/resolve', 2) + '?reference=' + encodeURIComponent(myLink))
        .then((response) => {
            reference = response.data.ocs.data.references[myLink]
        })

Then you can use the reference object you got:

.. code-block:: html

    <NcReferenceWidget :reference="reference" />

Register a reference provider
---------------------------

A reference provider is a class implementing the ``OCP\Collaboration\Reference\IReferenceProvider`` interface.
If you just want to resolve links, implement the ``IReferenceProvider`` interface.
This is described in the "Resolving links" section.

If you want your reference provider to be used by the Smart Picker, you need to extend the
``OCP\Collaboration\Reference\ADiscoverableReferenceProvider`` class to declare all required information.
There are 2 ways to appear in the smart picker.

* Either your reference provider implements the
``OCP\Collaboration\Reference\ISearchableReferenceProvider`` interface and you declare a list of unified search providers
that will be used in the Smart Picker
* or you don't implement this ``ISearchableReferenceProvider`` interface and make sure you register a custom picker component in the frontend.
This is described later in this documentation.

Link previews
---------------------------

Links that are not matched by any reference provider will always be handled by the OpenGraph provider as a fallback.
This provider will try to get the information declared in the target page. The link preview will be rendered with the
default widget.

For your provider to properly handle some links,
you need to implement the ``matchReference`` and ``resolve`` methods of ``IReferenceProvider``.

Match links
~~~~~~~~~~~~~~~~~~

The ``matchReference`` method of ``IReferenceProvider`` is used to know if a provider supports a link or not.

.. code-block:: php

    public function matchReference(string $referenceText): bool {
        // support all URLs starting with https://my.website.org
        return str_starts_with($referenceText, 'https://my.website.org/');
    }

Resolving links
~~~~~~~~~~~~~~~~~~

The ``resolve`` method of ``IReferenceProvider`` is used to get information about a link and return it in a structured
way via a ``OCP\Collaboration\Reference\Reference`` object.

Using the default widget
^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you are fine with the default widget rendering (image on the left, text and subtext on the right),
then you just need to provide a title, a description and optionally an image.

.. code-block:: php

    public function resolveReference(string $referenceText): ?IReference {
        if ($this->matchReference($referenceText)) {
            $title = $this->myAwesomeService->getLinkTitle($referenceText);
            $description = $this->myAwesomeService->getLinkDescription($referenceText);
            $imageUrl = $this->myAwesomeService->getImageUrl($referenceText);

            $reference = new Reference($referenceText);
            $reference->setTitle($title);
            $reference->setDescription($description);
            $reference->setImageUrl($imageUrl);
            return $reference;
        }
        return null;
    }

Using custom reference widgets
^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can customize the rendering of the links you support with your provider.

On the provider side, you need to pass all the information needed by your
custom reference widget component by setting the "rich object" of the ``Reference``
object returned by the ``resolve`` method.

.. code-block:: php

    public function resolveReference(string $referenceText): ?IReference {
        if ($this->matchReference($referenceText)) {
            $title = $this->myAwesomeService->getLinkTitle($referenceText);
            $description = $this->myAwesomeService->getLinkDescription($referenceText);
            $imageUrl = $this->myAwesomeService->getImageUrl($referenceText);
            $extraInformation = $this->myAwesomeService->getExtraInformation($referenceText);

            $reference = new Reference($referenceText);
            $reference->setRichObject(
                'my_rich_object_type',
                [
                    'title' => $title,
                    'description' => $description,
                    'image_url' => $imageUrl,
                    'extra_info' => $extraInformation,
                ]
            );

            return $reference;
        }
        return null;
    }

On the frontend side you need to implement and register your custom component. Here is a component example:

You need to react to the ``OCP\Collaboration\Reference\RenderReferenceEvent``
event to inject a script that will actually register the widget component.
For example, in your ``lib/AppInfo/Application.php`` file:

.. code-block:: php

    $context->registerEventListener(OCP\Collaboration\Reference\RenderReferenceEvent::class, MyReferenceListener::class);

The corresponding ``MyReferenceListener`` class can look like:

.. code-block:: php

    <?php
    namespace OCA\MyApp\Listener;

    use OCA\MyApp\AppInfo\Application;
    use OCP\Collaboration\Reference\RenderReferenceEvent;
    use OCP\EventDispatcher\Event;
    use OCP\EventDispatcher\IEventListener;
    use OCP\Util;

    class MyReferenceListener implements IEventListener {
        public function handle(Event $event): void {
            if (!$event instanceof RenderReferenceEvent) {
                return;
            }

            Util::addScript(Application::APP_ID, 'myapp-reference');
        }
    }

The ``myapp-reference.js`` file contains the widget registration:

.. code-block:: javascript

    import { registerWidget } from '@nextcloud/vue-richtext'
    import Vue from 'vue'
    import MyCustomWidgetComponent from './MyCustomWidgetComponent.vue'

    Vue.mixin({ methods: { t, n } })

    // here we register the MyCustomWidgetComponent to handle rich objects which type is 'my_rich_object_type'
    registerWidget('my_rich_object_type', (el, { richObjectType, richObject, accessible }) => {
        const Widget = Vue.extend(MyCustomWidgetComponent)
        new Widget({
            propsData: {
                richObjectType,
                richObject,
                accessible,
            },
        }).$mount(el)
    })

And last but not least, the MyCustomWidgetComponent Vue component:

.. code-block:: html

    <template>
        <div v-if="richObject">
            <div>
                <label>
                    {{ t('myapp', 'Title' }}
                </label>
                <span>
                    {{ richObject.title }}
                </span>
            <div>
            <div>
                <label>
                    {{ t('myapp', 'Extra info' }}
                </label>
                <span>
                    {{ richObject.extra_info }}
                </span>
            <div>
        </div>
    </template>

    <script>
    export default {
        name: 'MyCustomWidgetComponent',
        props: {
            richObjectType: {
                type: String,
                default: '',
            },
            richObject: {
                type: Object,
                default: null,
            },
            accessible: {
                type: Boolean,
                default: true,
            },
        },
    }
    </script>


Smart Picker
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
~~~~~~~~~~~~~~~~~~

If you want your reference provider to let users pick links from unified search results, your reference provider must
implement `OCP\\Collaboration\\Reference\\ISearchableReferenceProvider` and define the `getSupportedSearchProviderIds`
method which return a list of supported search provider IDs.

Once this provider is selected in the link picker, users will see a generic search interface giving results from
all the search providers you declared as supported. Once a result is selected, the link picker will return
the associated resource URL.

Register a custom picker component
~~~~~~~~~~~~~~~~~~

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
