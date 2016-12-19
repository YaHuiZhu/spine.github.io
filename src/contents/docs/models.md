---
title: Models
template: docs.jade
---

One of the challenges with moving state to the client-side is data management. Data management in stateful JavaScript applications is a completely different matter to how it's normally handled in conventional server side apps. There’s no request/response model, and you don’t have access to server-side variables. Instead, data is fetched remotely and stored temporarily on the client-side. This has the advantage that data access is immediate, and users are rarely, if ever, left waiting for remote data to load.

After the initial page load, remote data is stored locally in class structures called models. Models are the core to Spine, and absolutely critical to your applications. Not only do they store all the application's data, but they are also where any logic associated with that data is kept.

Models should be de-coupled from the rest of your application, and completely independent. Model data can be persisted with [HTML5 Local Storage](local.html) or [Ajax](ajax.html).

## Implementation

Models are created by extending `Spine.Model`:

    //= CoffeeScript
    class Contact extends Spine.Model
      @configure "Contact", "first_name", "last_name"

    //= JavaScript
    var Contact = Spine.Model.sub();
    Contact.configure("Contact", "first_name", "last_name");

You should call `configure()` before anything else inside the model, since it bootstraps various variables and events. Pass `configure()` the model name, and any attributes the model has.

Models are like any other CoffeeScript class, so you can add class/instance methods as usual:

    //= CoffeeScript
    class Contact extends Spine.Model
      @configure "Contact", "first_name", "last_name"

      @filter: (query) ->
        @select (c) ->
          c.first_name.indexOf(query) isnt -1

      fullName: -> [@first_name, @last_name].join(' ')

    //= JavaScript
    var Contact = Spine.Model.sub();
    Contact.configure("Contact", "first_name", "last_name");

    Contact.extend({
      filter: function(query) {
        return this.select(function(c){
          return c.first_name.indexOf(query) != -1
        });
      }
    });

    Contact.include({
      fullName: function(){
        return(this.first_name + " " + this.last_name);
      }
    });

Models are Spine [modules](classes.html), so you can treat them as such, extending and including properties.

    //= CoffeeScript
    class Contact extends Spine.Model
      @configure "Contact", "first_name", "last_name"

      @extend MyModule

    //= JavaScript
    var Contact = Spine.Model.sub();
    Contact.configure("Contact", "first_name", "last_name");

    Contact.extend(MyModule);

Model instances are created with `new`, passing in an optional set of attributes.

    //= CoffeeScript
    contact = new Contact(first_name: "Alex", last_name: "MacCaw")
    assertEqual( contact.fullName(), "Alex MacCaw" )

    //= JavaScript
    var contact = new Contact({first_name: "Alex", last_name: "MacCaw"});
    assertEqual( contact.fullName(), "Alex MacCaw" );

Models can be also be easily subclassed:

    //= CoffeeScript
    class User extends Contact
      @configure "User"

    //= JavaScript
    var User = Contact.sub();
    User.configure("User");

## Saving/Retrieving Records

Once an instance is created it can be saved in memory by calling `save()`.

    //= CoffeeScript
    class Contact extends Spine.Model
      @configure "Contact", "first_name", "last_name"

    contact = new Contact(first_name: "Joe")
    contact.save()

    //= JavaScript
    var Contact = Spine.Model.sub();
    Contact.configure("Contact", "first_name", "last_name");

    var contact = new Contact({first_name: "Joe"});
    contact.save();

When a record is saved, Spine automatically creates a simple client ID if it doesn't already exist.

    assertEqual( contact.id, "c-1" )

You can use this ID to retrieve the saved record using `find()`.

    identicalContact = Contact.find( contact.id )
    assert( contact.eql( identicalContact ) )

If `find()` fails to retrieve a record the default behaviour is for null to be returned. it up to you to check if anything is returned. alternately you can check for the existence of records by calling `exists()`.

    assert( Contact.exists( contact.id ) )

You also have the option to define a custom `notFound()` method on your Model. This method is called in the event find fails for the id it was given. You could have notFound() throw an exception (to mimic behavior of older versions of Spine) or you may want check a remote data source for the record.

    //=CoffeeScript
    Class Contact extends Spine.Model
      ...

      notFound: (missingId)->
        @one 'ajaxSuccess', ->
          alert('found that one after all')
        @fetch(id:missingId)
        # actually would be good to write custom ajax that returns a promise object

Once you've changed any of a record's attributes, you can update it in-memory by re-calling `save()`.

    //= CoffeeScript
    contact = Contact.create(first_name: "Polo")
    contact.save()
    contact.first_name = "Marko"
    contact.save()

You can also use `first()` or `last()` on the model to retrieve the first and last records respectively.

    //= CoffeeScript
    firstContact = Contact.first()

`first()` or `last()` also accept a number of records

    //= CoffeeScript
    recentMessages = Message.last(5)

To retrieve every contact, use `all()`.

    //= CoffeeScript
    contacts = Contact.all()
    console.log(contact.name) for contact in contacts

Spine mainains the order of your model collection as well so if you wanted a sort of paginated list you might use `splice()` which follows the javascript pattern

    //= CoffeeScript
    allUsersExceptFirst3 = User.slice(3)
    users7through13 = User.slice(6,13)

You can pass a function that'll be iterated over every record using `each()`.

    //= CoffeeScript
    Contact.each (contact) -> console.log(contact.first_name)

Or select a subset of records with `select()`.

    //= CoffeeScript
    Contact.select (contact) -> contact.first_name

## Validation

Validating models is dirt simple, simply override the `validate()` function with your own custom one.

    //= CoffeeScript
    class Contact extends Spine.Model
      validate: ->
        unless @first_name
          "First name is required"

    //= JavaScript
    var Contact = Spine.Model.sub();
    Contact.configure("Contact", "first_name", "last_name");
    Contact.include({
      validate: function(){
        if ( !this.first_name )
          return "First name is required";
      }
    });

If `validate()` returns anything, the validation will fail and an *error* event will be fired on the model. You can catch this by listening for it on the model, notifying the user.

    //= CoffeeScript
    Contact.bind "error", (rec, msg) ->
      alert("Contact failed to save - " + msg)

In addition, `save()`, `create()` and `updateAttributes()` will all return false if validation fails. For more information about validation, see the [validation guide](forms.html).

## Serialization

Spine's models include special support for JSON serialization. To serialize a record, call `JSON.stringify()` passing the record, or to serialize every record, pass the model.

    JSON.stringify(Contact)
    JSON.stringify(Contact.first())

Alternatively, you can retrieve an instance's attributes and implement your own serialization by calling `attributes()`.

    //= CoffeeScript
    contact = new Contact(first_name: "Leo")
    assertEqual( contact.attributes(), {first_name: "Leo"} )

    Contact.include
      toXML: ->
        serializeToXML(@attributes())

If you're using an older browser which doesn't have native JSON support (i.e. IE 7), you'll need to include [json2.js](https://github.com/douglascrockford/JSON-js/blob/master/json2.js) which adds legacy support.

## Persistence

While storing records in memory is useful for quick retrieval, persisting them in one way or another is often required. Spine includes a number of pre-existing storage modules, such as Ajax and HTML5 Local Storage, which you can use for persistence. Please check out the [Ajax](ajax.html) and [Local Storage guides](local.html) for more information.

If you want to pull from one of these sources to populate your model collection use `fetch()`

    Contact.fetch()

## Events

You've already seen that models have some events associated with them, such as *error* and *ajaxError*, but what about callbacks to create/update/destroy operations? Well, conveniently Spine includes those too, allowing you to bind to the following events:

* *save* - record was saved (either created/updated)
* *update* - record was updated
* *create* - record was created
* *destroy* - record was destroyed
* *change* - any of the above, record was created/updated/destroyed
* *refresh* - when records are invalidated or appended using the refresh method
* *error* - validation failed

For example, you can bind to a model's *create* event like so:

    //= CoffeeScript
    Contact.bind "create", (newRecord) ->
      # New record was created

For model level callbacks, any associated record is always passed to the callback. The other option is to bind to the event directly on the record.

    //= CoffeeScript
    contact = Contact.first()
    contact.bind "save", ->
      # Contact was updated
      updateInterface()

The callback's context will be the record that the event listener was placed on. You'll find model events crucial when it comes to binding records to the view, making sure the view is kept in sync with your application's data.

If you want to remove events, you can unbind specific events by calling `unbind()` on the Model. See the [event documentation](events.html) for more information on how you should use `unbind()`. Model instances also have an `unbind()` function, but it can only be used to remove every event listener associated with that instance.

## Dynamic records

One rather neat addition to Spine's models is dynamic records, which use prototypal inheritance to stay updated. Any calls to `find()`, `all()`, `first()`, `last()` etc, and model event callbacks return a *clone* of the saved record. This means that whenever a record is updated, all of its clones will immediately reflect that update.

Let's give you a code example; we're going to create an asset, and a clone of that asset. You'll notice that when the asset is updated, the clone has also automatically changed.

    //= CoffeeScript
    asset = Asset.create(name: "whatshisname")

    # Completely new asset instance
    clone = Asset.find(asset.id)

    # Change saved asset
    asset.updateAttributes(name: "bob")

    # Clone reflects changes
    assertEqual(clone.name, "bob")

This means that you never have to bother calling some sort of `reload()` functions on instances. You can be sure that all instances are constantly in sync with their saved versions.

## Relationships

Spine's relationship module (spine.relation) adds has support for *has-many*, *has-one* and *belongs-to* model relationships. For more information, see the [relationships guide](relations.html).

## API documentation

For more information about models, please see the [full API documentation](models.html).
