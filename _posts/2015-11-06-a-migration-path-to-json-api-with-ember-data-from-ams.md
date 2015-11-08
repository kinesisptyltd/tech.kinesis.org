---
layout: post
title: A migration path to JSON API with Ember Data from AMS
tags: emberjs jsonapi active-model-serializers rails jsonapi-resources
authors: david chris
---

We've been using Ember.js for about a year now, and as is typical with Rails
shops we began our journey with ActiveModelSerializers to drive our API.
As our Ember application grew, we quickly discovered problems with the way AMS
approaches serialization.

Some of the issues we ran into with AMS are:

  - Inability to handle requests for specific fields from the client. Eg. only
    return attributes a and b instead of a, b and c.
  - Sideloading models is per serializer and when there is a large number of
    related models, you often end up overfetching data

For example, given the following serializers...

```ruby
class PostSerializer < ActiveModel::Serializer
  has_many :comments, embed_in_root: true
end

class CommentSerializer < ActiveModel::Serializer
end
```

...we would not be able to only get the posts without always returning the comment
data. This quickly becomes untenable with a large number of related models. We had
a number of ways to work around these problems (e.g. action specific serializers) but
none of them were satisfactory.

With the release of Ember 2.0 and the transition to JSON API as the default adapter, we
were keen to take advantage of the features it provides. However, we quickly discovered that
our app was too large and too complex to do a full transition in a timeframe that fit
our business requirements. We basically wanted to change as many small, isolated bits as we could,
but still leave some of the meatier bits alone until we have more time to tackle them.

This wasn't a big deal for some parts of the site. For many of the Rails models
it was as simple as deleting the serializer, and replacing it with a Resource class from
the [jsonapi-resources](https://github.com/cerebris/jsonapi-resources) gem.
But for one of our god models (`asset`) that has whole bunch of related records,
we ended up having to maintain two Ember stores - AMS and JSON API - simultaneously, and handle
the requests in the Rails controller differently depending on which store was requesting it.
And since we couldn't find anyone else who'd tackled the same problem, we had to go it alone.

So, how did we do it?

## Ember

First, we realised that we'd need a new store, as well as a new JSON API adapter and serializer for our `asset` model.
The adapter and serializer would need to be the JSON API ones instead of the AMS ones:

```javascript
// app/adapters/jsonapi-asset.js
import Ember from 'ember';
import DS from 'ember-data';

export default DS.JSONAPIAdapter;
```

```javascript
// app/serializers/jsonapi-asset.js
import Ember from 'ember';
import DS from 'ember-data';

export default DS.JSONAPISerializer;
```

The interesting part is the new store. It would still need basic store functionality,
but would need to call out to our new adapter and serializer. This is where the magic happens:

```javascript
// app/services/jsonapi-store.js
import Store from 'integrated/services/store'; // the regular DS.Store

export default Store.extend({
  lookupAdapter(name) {
    if (name === 'asset') {
      return this._super(`jsonapi-${name}`);
    }
    return this._super(name);
  },

  lookupSerializer(name, fallbacks) {
    if (name === 'asset') {
      return this._super(`jsonapi-${name}`, fallbacks);
    }
    return this._super(...arguments);
  }
});
```

With these overridden methods, when we do a `store.query('asset')`,
the store calls out to our above-mentioned `jsonapi-asset`
adapter and serializer, but querying any other model falls through to the old AMS ones.

Great! So we're now sending off JSON API requests, but we're not going to
 be able to use this store until we inject it where it's needed.
For that, we'll use a new initializer:

```javascript
// app/initializers/jsonapi-store.js
export default {
  name: 'jsonapi-store',
  after: 'store',

  initialize(registry, application) {
    // any paths where you want 'store' to refer to the new jsonapi-store
    const paths = [
      'route:posts',
      'controller:posts'
    ];

    // inject jsonapi-store as 'store'
    paths.forEach((path) => {
      application.inject(path, 'store', 'service:jsonapi-store');
    });


    // in this case, we're injecting jsonapi-store with a different name
    application.inject('controller:assets', 'jsonapiStore', 'service:jsonapi-store');
  }
};
```

Now, `store` means the jsonapi-store in the `posts` route and controller,
but in `controller:assets`, we have both `store` and `jsonapiStore`. Awesome!

## Rails

On the Rails side, we need an `asset` resource as per the gem instructions, but we also need
the controller to be able to handle both AMS style and JSONAPI style requests.
Usually with [jsonapi-resources](https://github.com/cerebris/jsonapi-resources), one can get away with
just inheriting from the `JSONAPI::ResourceController`, but in our case we're going to need to mix in the functionality
while still keeping the old methods around for the AMS style requests. Thankfully we can just `include JSONAPI::ActsAsResourceController` for that.

We're also going to need a callback to check which kind of request we're receiving, and route to the correct handler.

Lastly, we need to skip some of the gem callbacks that will prevent us from being able to use both kinds of requests.

```ruby
# app/controllers/assets_controller.rb

class AssetsController < BaseController
  include JSONAPI::ActsAsResourceController # mix in the functionality

  before_action :route_to_jsonapi, if: :use_jsonapi?
  skip_before_action :setup_request # we'll perform this manually if it's JSONAPI
  skip_before_action :ensure_correct_media_type

  def use_jsonapi?
    # Ember sends different headers when using JSONAPI than with AMS,
    # so we can use this fact to determine which kind of request we're receiving
    request.content_type == JSONAPI::MEDIA_TYPE || request.headers["ACCEPT"] == JSONAPI::MEDIA_TYPE
  end

  def route_to_jsonapi
    setup_request # do it manually and...
    process_request_operations # let the gem take over control for JSONAPI
    false
  end

  # rest of the methods (index, show etc.) here
end
```

Also, don't forget to add the routes:

```ruby
  jsonapi_resources :assets do
    jsonapi_relationships
  end
```

Now any requests that come in with JSONAPI will be handled by the gem and go through the corresponding `JSONAPI::Resource`.
AMS ones can still hit the regular index, show, or even custom methods.

### Relationships in Rails

Importantly, if you want to be able to use the really handy sideloading feature of JSONAPI, you'll need to make sure you create `JSONAPI::Resource`s for your related records that includes the relationship:

```ruby
class CommentResource < JSONAPI::Resource
  attributes :body

  has_one :post # <-- like this.
end
```

Then, as long as you have the `jsonapi_relationships` in the route, you can use `include` to sideload records:

```javascript
this.jsonapiStore.query('comment', {
  include: 'post'
}).then((result) => {
  // ...
});
```

That should be all you need to maintain two simultaneous stores in Ember and Rails. Hopefully we haven't forgotten anything. If we have, feel free to ask a question or leave a comment.
