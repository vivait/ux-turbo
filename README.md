# Symfony UX Turbo

Symfony UX Turbo is a Symfony bundle integrating the [Hotwire Turbo](https://turbo.hotwire.dev)
library in Symfony applications. It is part of [the Symfony UX initiative](https://symfony.com/ux).

Symfony UX Turbo allows having the same user experience as with [Single Page Apps](https://en.wikipedia.org/wiki/Single-page_application)
but without having to write a single line of JavaScript!

Symfony UX Turbo also integrates with [Symfony Mercure](https://symfony.com/doc/current/mercure.html)
or any other transports to broadcast DOM changes to all currently connected users!

You're in a hurry? Take a look at [the chat example](#sending-async-changes-using-mercure-a-chat)
to discover the full potential of Symfony UX Turbo.

## Installation

Symfony UX Turbo requires PHP 7.2+ and Symfony 5.2+.

Install this bundle using Composer and Symfony Flex:

```sh
composer require symfony/ux-turbo

# Don't forget to install the JavaScript dependencies as well and compile
yarn install --force
yarn encore dev
```

## Usage

### Accelerating Navigation with Turbo Drive

Turbo Drive enhances page-level navigation. It watches for link clicks and form submissions,
performs them in the background, and updates the page without doing a full reload.
This gives you the "single-page-app" experience without major changes to your code!

Turbo Drive is automatically enabled when you install Symfony UX Turbo. And while
you don't need to make major changes to get things to work smoothly, there are 3
things to be aware of:

#### 1. Make sure your JavaScript is Turbo-ready

Because navigation no longer results in full page refreshes, you may need to
adjust your JavaScript to work properly. The best solution is to write your
JavaScript using [Stimulus](https://stimulus.hotwire.dev/) or something similar.

We also recommend that you place your `script` tags live inside your `head` tag so
that they aren't reloaded on every navigation (Turbo re-executes any `script` tags
inside `body` on every navigation). Add a `defer` attribute to each `script` tag
to prevent it from blocking the page load. See
[Moving <script> inside <head> and the "defer" Attribute](https://symfony.com/blog/moving-script-inside-head-and-the-defer-attribute)
for more info.

#### 2. Reloading When a JavaScript/CSS File Changes

Turbo drive can automatically perform a full refresh if the content of one of
your CSS or JS files _changes_, to ensure that your users always have the latest
version.

To enable this, first verify that you have versioning enabled in Encore so that
your filenames change when the file contents change:

```js
// webpack.config.js

Encore.
    // ...
    .enableVersioning(Encore.isProduction())
```

Then add a `data-turbo-track="reload"` attribute to all of your `script` and
`link` tags:

```yml
# config/packages/webpack_encore.yaml
webpack_encore:
    # ...

    script_attributes:
        defer: true
        'data-turbo-track': reload
    link_attributes:
        'data-turbo-track': reload
```

For more info, see:
[Turbo: Reloading When Assets Change](https://turbo.hotwire.dev/handbook/drive#reloading-when-assets-change)

#### 3. Form Response Code Changes

Turbo Drive also converts form submissions to AJAX calls. To get it to work, you
_do_ need to adjust your code to return a 422 status code on a validation error
(instead of a 200).

If you're using Symfony 5.3, the new `handleForm()` shortcut takes care of this
automatically:

```php
/**
 * @Route("/product/new", name="product_new")
 */
public function newProduct(Request $request): Response
{
    return $this->handleForm(
        $this->createForm(ProductFormType::class, null, [
            'action' => $this->generateUrl('product_new'),
        ]),
        $request,
        function (FormInterface $form) {
            // save...

            return $this->redirectToRoute(
                'product_list',
                [],
                Response::HTTP_SEE_OTHER
            );
        },
        function (FormInterface $form) {
            return $this->render('product/new.html.twig', [
                'form' => $form->createView(),
            ]);
        }
    );
}
```

If you're _not_ using the `handleForm()` shortcut, adjust your code manually:

```diff
/**
 * @Route("/product/new")
 */
public function newProduct(Request $request): Response
{
    $form = $this->createForm(ProductFormType::class);
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        // save...

-        return $this->redirectToRoute('product_list');
+        return $this->redirectToRoute('product_list', [], Response::HTTP_SEE_OTHER);
    }

+    $response = new Response(null, $form->isSubmitted() ? 422 : 200);

    return $this->render('product/new.html.twig', [
        'form' => $form->createView()
-    ]);
+    ], $response);
}
```

This changes the response status code to 422 on validation error, which tells Turbo
Drive that the form submit failed and it should re-render with the errors. This
_also_ changes the redirect status code from 302 (the default) to 303
(`HTTP_SEE_OTHER`). That's not required for Turbo Drive, but 303 is "more correct"
for this situation.

> **NOTE:**
> When your form contains more than one submit button and, you want to check which of the buttons was clicked
> to adapt the program flow in your controller. You need to add a value to each button because
> Turbo Drive doesn't send element with empty value:

```php
$builder
    // ...
    ->add('save', SubmitType::class, [
        'label' => 'Create Task',
        'attr' => [
            'value' => 'create-task'
        ]
    ])
    ->add('saveAndAdd', SubmitType::class, [
        'label' => 'Save and Add',
        'attr' => [
            'value' => 'save-and-add'
        ]
    ]);
```

#### More Turbo Drive Info

[Read the Turbo Drive documentation](https://turbo.hotwire.dev/handbook/drive) to learn about the advanced features offered
by Turbo Drive.

### Decomposing Complex Pages with Turbo Frames

Once Symfony UX Turbo is installed, you can also leverage [Turbo Frames](https://turbo.hotwire.dev/handbook/introduction#turbo-frames-decompose-complex-pages):

```twig
{# home.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
    <turbo-frame id="the_frame_id">
        <a href="{{ path('another-page') }}">This block is scoped, the rest of the page will not change if you click here!</a>
    </turbo-frame>
{% endblock %}
```

```twig
{# another-page.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
    <div>This will be discarded</div>

    <turbo-frame id="the_frame_id">
        The content of this block will replace the content of the Turbo Frame!
        The rest of the HTML generated by this template (outside of the Turbo Frame) will be ignored.
    </turbo-frame>
{% endblock %}
```

The content of a frame can be lazy loaded:

```twig
{# home.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
    <turbo-frame id="the_frame_id" src="{{ path('block') }}">
        A placeholder.
    </turbo-frame>
{% endblock %}
```

In your controller, you can detect if the request has been triggered by a Turbo Frame, and retrieve the ID of this frame:

```php
// src/Controller/MyController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

class MyController
{
    #[Route('/')]
    public function home(Request $request): Response
    {
        // Get the frame ID (will be null if the request hasn't been triggered by a Turbo Frame)
        $frameId = $request->headers->get('Turbo-Frame');

        // ...
    }
}
```

#### Writing Tests

Under the hood, Symfony UX Turbo relies on JavaScript to update the HTML page.
To test if your website works properly, you will have to write [UI tests](https://martinfowler.com/articles/practical-test-pyramid.html#UiTests).

Fortunately, we've got you covered! [Symfony Panther](https://github.com/symfony/panther) is a convenient testing tool
using real browsers to test your Symfony application. It shares the same API as BrowserKit, the functional testing tool shipped with Symfony.

[Install Symfony Panther](https://github.com/symfony/panther#installing-panther), and write a test for our Turbo Frame:

```php
// tests/TurboFrameTest.php
namespace App\Tests;

use Symfony\Component\Panther\PantherTestCase;

class TurboFrameTest extends PantherTestCase
{
    public function testFrame(): void
    {
        $client = self::createPantherClient();
        $client->request('GET', '/');

        $client->clickLink('This block is scoped, the rest of the page will not change if you click here!');
        $this->assertSelectorTextContains('body', 'This will replace the content of the Turbo Frame!');
    }
}
```

Run `bin/phpunit` to execute the test! Symfony Panther automatically starts your application with a web server
and tests it using Google Chrome or Firefox!

You can even watch changes happening in the browser by using: `PANTHER_NO_HEADLESS=1 bin/phpunit --debug`

[Read the Turbo Frames documentation](https://turbo.hotwire.dev/handbook/frames) to learn everything you can do using Turbo Frames.

### Coming Alive with Turbo Streams

Turbo Streams are a way for the server to send partial page updates to clients.
There are two main ways to receive the updates:

-   in response to a user action, for instance when the user submits a form;
-   asynchronously, by sending updates to clients using [Mercure](https://mercure.rocks),
    [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) and similar protocols.

#### Forms

Let's discover how to use Turbo Streams to enhance your [Symfony forms](https://symfony.com/doc/current/forms.html):

```php
// src/Controller/TaskController.php
namespace App\Controller;

// ...
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\UX\Turbo\Stream\TurboStreamResponse;

class TaskController extends AbstractController
{
    public function new(Request $request): Response
    {
        $task = new Task();

        $form = $this->createForm(TaskType::class, $task);
        $form->handleRequest($request);

        $submitted = $form->isSubmitted();
        $valid = $submitted && $form->isValid();

        if ($valid) {
            $task = $form->getData();
            // ... perform some action, such as saving the task to the database

            // 🔥 The magic happens here! 🔥
            if (TurboStreamResponse::STREAM_FORMAT === $request->getPreferredFormat()) {
                // If the request comes from Turbo, only send the HTML to update using a TurboStreamResponse
                return $this->render('task/success.stream.html.twig', ['task' => $task], new TurboStreamResponse());
            }

            // If the client doesn't support JavaScript, or isn't using Turbo, the form still works as usual.
            // Symfony UX Turbo is all about progressively enhancing your apps!
            return $this->redirectToRoute('task_success', [], Response::HTTP_SEE_OTHER);
        }

        // Symfony 5.3+
        return $this->renderForm('task/new.html.twig', $form);

        // Older versions
        $response = $this->render('task/new.html.twig', [
            'form' => $form->createView(),
        ]);
        if ($submitted && !$valid) {
            $response->setStatusCode(Response::HTTP_UNPROCESSABLE_ENTITY);
        }

        return $response;
    }
}
```

```twig
{# success.stream.html.twig #}

<turbo-stream action="replace" target="my_div_id">
    <template>
        The element having the id "my_div_id" will be replace by this block, without a full page reload!

        <div>The task "{{ task }}" has been created!</div>
    </template>
</turbo-stream>
```

Supported actions are `append`, `prepend`, `replace`, `update` and `remove`.
[Read the Turbo Streams documentation for more details](https://turbo.hotwire.dev/handbook/streams).

#### Sending Async Changes using Mercure: a Chat

Symfony UX Turbo also supports broadcasting HTML updates to all currently connected clients,
using the [Mercure](https://mercure.rocks) protocol or any other.

To illustrate this, let's build a chat system with **0 lines of JavaScript**!

Start by installing [the Mercure support](https://symfony.com/doc/current/mercure.html) on your project:

```sh
composer require symfony/ux-turbo-mercure
yarn install --force
yarn encore dev
```

The easiest way to have a working development (and production-ready) environment is to use [Symfony Docker](https://github.com/dunglas/symfony-docker/),
which comes with a Mercure hub integrated in the web server.

If you use Symfony Flex, the configuration has been generated for you, be sure to update the `MERCURE_URL` in
the `.env` file to point to a Mercure Hub (it's not necessary if you are using Symfony Docker).

Otherwise, configure Mercure Hub(s) to use:

```yaml
# config/packages/turbo.yaml
turbo:
    mercure:
        hubs: [default]
```

Let's create our chat:

```php
// src/Controller/ChatController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Mercure\PublisherInterface;

class ChatController extends AbstractController
{
    public function chat(Request $request, PublisherInterface $mercure): Response
    {
        $form = $this->createFormBuilder()
            ->add('message', TextType::class, ['attr' => ['autocomplete' => 'off']])
            ->add('send', SubmitType::class)
            ->getForm();

        $emptyForm = clone $form; // Used to display an empty form after a POST request
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $data = $form->getData();

            // 🔥 The magic happens here! 🔥
            // The HTML update is pushed to the client using Mercure
            $mercure->publish(new Update(
                'chat',
                $this->renderView('chat/message.stream.html.twig', ['message' => $data['message']])
            ));

            // Force an empty form to be rendered below
            // It will replace the content of the Turbo Frame after a post
            $form = $emptyForm;
        }

        return $this->render('chat/index.html.twig', [
            'form' => $form->createView(),
         ]);
    }
}
```

```twig
{# chat/index.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
    <h1>Chat</h1>

    <div id="messages" {{ turbo_stream_listen('chat') }}>
        {#
            The messages will be displayed here.
            "turbo_stream_listen()" automatically registers a Stimulus controller that subscribes to the "chat" topic as managed by the transport.
            All connected users will receive the new messages!
         #}
    </div>

    <turbo-frame id="message_form">
        {{ form(form) }}

        {#
            The form is displayed in a Turbo Frame, with this trick a new empty form is displayed after every post,
            but the rest of the page will not change.
        #}
    </turbo-frame>
{% endblock %}
```

```twig
{# chat/message.stream.html.twig #}

{# New messages received through the Mercure connection are appended to the div with the "messages" ID. #}
<turbo-stream action="append" target="messages">
    <template>
        <div>{{ message }}</div>
    </template>
</turbo-stream>
```

Keep in mind that you can use all features provided by Symfony Mercure, including [private updates](https://symfony.com/doc/current/mercure.html#authorization)
(to ensure that only authorized users will receive the updates) and [async dispatching with Symfony Messenger](https://symfony.com/doc/current/mercure.html#async-dispatching).

#### Broadcast Doctrine Entities Update

Symfony UX Turbo also comes with a convenient integration with Doctrine ORM.

With a single attribute, your clients can subscribe to creations, updates and deletions of entities:

```php
// src/Entity/Book.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\UX\Turbo\Attribute\Broadcast;

/**
 * @ORM\Entity
 */
#[Broadcast] // 🔥 The magic happens here
class Book
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    public ?int $id = null;

    /**
     * @ORM\Column
     */
    public string $title = '';
}
```

To subscribe to updates of an entity, pass it as parameter of the `turbo_stream_listen()` Twig helper:

```twig
<div id="book_{{ book.id }}" {{ turbo_stream_listen(book) }}></div>
```

Alternatively, you can subscribe to updates made to all entities of a given class by using its Fully Qualified Class Name:

```twig
<div id="books" {{ turbo_stream_listen('App\\Entity\\Book') }}></div>
```

Finally, create the template that will be rendered when an entity is created, modified or deleted:

```twig
{# templates/broadcast/Book.stream.html.twig #}

{% block create %}
    <turbo-stream action="append" target="books">
        <template>
            <div id="{{ 'book_' ~ id }}">{{ entity.title }} (#{{ id }})</div>
        </template>
    </turbo-stream>
{% endblock %}

{% block update %}
    <turbo-stream action="update" target="book_{{ id }}">
        <template>
            {{ entity.title }} (#{{ id }}, updated)
        </template>
    </turbo-stream>
{% endblock %}

{% block remove %}
    <turbo-stream action="remove" target="book_{{ id }}"></turbo-stream>
{% endblock %}
```

By convention, Symfony UX Turbo will look for a template named `templates/broadcast/{ClassName}.stream.html.twig`.
This template **must** contain at least 3 blocks: `create`, `update` and `remove` (they can be empty, but they must exist).

Every time an entity marked with the `Broadcast` attribute changes, Symfony UX Turbo will render the associated template
and will broadcast the changes to all connected clients.

Each block must contain a list of Turbo Stream actions. These actions will be automatically applied by Turbo to the DOM
tree of every connected client. Each template can contain as many actions as needed.

For instance, if the same entity is displayed on different pages, you can include all actions updating these different places
in the template.
Actions applying to non-existing DOM elements will simply be ignored.

The current entity, the string representation of its identifier(s), the action (`create`, `update` or `remove`) and options set on the `Broadcast` attribute
are passed to the template as variables: `entity`, `id`, `action` and `options`.

### Broadcast Conventions and Configuration

Because Symfony UX Turbo needs access to their identifier, entities have to either be managed
by Doctrine ORM, have a public property named `id`, or have a public method named `getId()`.

Symfony UX Turbo will look for a template named after mapping their Fully Qualified Class Names.
For example and by default, if a class marked with the `Broadcast` attribute is named `App\Entity\Foo`,
the corresponding template will be found in `templates/broadcast/Foo.stream.html.twig`.

It's possible to configure own namespaces are mapped to templates by using the `turbo.broadcast.entity_template_prefixes` configuration options.
The default is defined as such:

```yaml
# config/packages/turbo.yaml
turbo:
    broadcast:
        entity_template_prefixes:
            App\Entity\: broadcast/
```

Finally, it's also possible to explicitly set the template to use with the `template` parameter of the `Broadcast` attribute:

```php
#[Broadcast(template: 'my-template.stream.html.twig')]
class Book { /* ... */ }
```

### Broadcast Options

The `Broadcast` attribute comes with a set of handy options:

-   `transports` (`string[]`): a list of transports to broadcast to
-   `topics` (`string[]`): a list of topics to use, the default topic is derived from the FQCN of the entity and from its id
-   `template` (`string`): Twig template to render (see above)

Options are transport-sepcific.
When using Mercure, some extra options are supported:

-   `private` (`bool`): marks Mercure updates as private
-   `sse_id` (`string`): `id` field of the SSE
-   `sse_type` (`string`): `type` field of the SSE
-   `sse_retry` (`int`): `retry` field of the SSE

Example:

```php
// src/Entity/Book.php
namespace App\Entity;

use Symfony\UX\Turbo\Attribute\Broadcast;

#[Broadcast(template: 'foo.stream.html.twig', private: true)]
class Book
{
    // ...
}
```

### Using Multiple Transports

Symfony UX Turbo allows sending Turbo Streams updates using multiple transports.
For instance, it's possible to use several Mercure hubs with the following configuration:

```yaml
# config/packages/mercure.yaml
mercure:
    hubs:
        hub1:
            url: https://hub1.example.net/.well-known/mercure
            jwt: snip
        hub2:
            url: https://hub2.example.net/.well-known/mercure
            jwt: snip
```

```yaml
# config/packages/turbo.yaml
turbo:
    mercure:
        hubs: [hub1, hub2]
```

Use the appropriate Mercure `HubInterface` service to send a change using a specific transport:

```php
// src/Controller/MyController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\Update;

class MyController extends AbstractController
{
    public function publish(HubInterface $hub1): Response
    {
        $id = $hub1->publish(new Update('topic', 'content'));

        return new Response("Update #{$id} published.");
    }
}
```

Changes made to entities marked with the `#[Broadcast]` attribute will be sent using all configured transport by default.
You can specify the list of transports to use for a specific entity class using the `transports` parameter:

```php
// src/Entity/Book.php
namespace App\Entity;

use Symfony\UX\Turbo\Attribute\Broadcast;

#[Broadcast(transports: ['hub1', 'hub2'])]
/** ... */
class Book
{
    // ...
}
```

Finally, generate the HTML attributes registering the Stimulus controller
corresponding to your transport by passing an extra argument to `turbo_stream_listen()`:

```twig
<div id="messages" {{ turbo_stream_listen('App\Entity\Book', 'hub2') }}></div>
```

### Registering a Custom Transport

If you prefer using another protocol than Mercure, you can create custom transports:

```php
// src/Turbo/Broadcaster.php
namespace App\Turbo;

use Symfony\UX\Turbo\Attribute\Broadcast;
use Symfony\UX\Turbo\Broadcaster\BroadcasterInterface;

class Broadcaster implements BroadcasterInterface
{
    public function broadcast(object $entity, string $action): void
    {
        // This method will be called everytime an object marked with the #[Broadcast] attribute is changed
        $attribute = (new \ReflectionClass($entity))->getAttributes(Broadcast::class)[0] ?? null;
        // ...
    }
}
```

```php
// src/Turbo/TurboStreamListenRenderer.php
namespace App\Turbo;

use Symfony\Component\DependencyInjection\Attribute\AsTaggedItem;
use Symfony\UX\Turbo\Twig\TurboStreamListenRendererInterface;
use Symfony\WebpackEncoreBundle\Twig\StimulusTwigExtension;
use Twig\Environment;

#[AsTaggedItem(index: 'my-transport')]
class TurboStreamListenRenderer implements TurboStreamListenRendererInterface
{
    public function __construct(
        private StimulusTwigExtension $stimulusTwigExtension,
    ) {}

    /**
     * @param string|object $topic
     */
    public function renderTurboStreamListen(Environment $env, $topic): string
    {
        return $this->stimulusTwigExtension->renderStimulusController(
            $env,
            'your_stimulus_controller',
            [/* controller values such as topic */]
        );
    }
}
```

The broadcaster must be registered as a service tagged with `turbo.broadcaster` and the renderer must be tagged with `turbo.renderer.stream_listen`.
If you enabled [autoconfigure option](https://symfony.com/doc/current/service_container.html#the-autoconfigure-option) (it's the case by default), these tags will be added automatically because these classes implement the `BroadcasterInterface` and `TurboStreamListenRendererInterface` interfaces,
the related services will be.

## Backward Compatibility promise

This bundle aims at following the same Backward Compatibility promise as the Symfony framework:
[https://symfony.com/doc/current/contributing/code/bc.html](https://symfony.com/doc/current/contributing/code/bc.html)

However, it is currently considered
[**experimental**](https://symfony.com/doc/current/contributing/code/experimental.html),
meaning it is not bound to Symfony's BC policy for the moment.

## Credits

Symfony UX Turbo has been created by [Kévin Dunglas](https://dunglas.fr).
It has been inspired by [hotwired/turbo-rails](https://github.com/hotwired/turbo-rails)
and [sroze/live-twig](https://github.com/sroze/live-twig).
