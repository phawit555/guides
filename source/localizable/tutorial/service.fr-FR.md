For Super Rentals, we want to be able to display a map showing where each rental is. To implement this feature, we will take advantage of several Ember concepts:

  1. A utility function to create a map from the Google Maps API.
  2. A service to keep a cache of rendered maps to use in different places in the application.
  3. A component to display a map on each rental listing.

### Making Google Maps Available

Before implementing a map, we need to make a 3rd party map API available to our Ember app. There are several ways to include 3rd party libraries in Ember. See the guides section on [managing dependencies](../../addons-and-dependencies/managing-dependencies/) as a starting point when you need to add one.

Since Google provides its map API as a remote script, we'll use curl to download it into our project's vendor directory.

From your project's root directory, run the following command to put the Google maps script in your projects vendor folder as `gmaps.js`. `Curl` is a UNIX command, so if you are on windows you should take advantage of [Windows bash support](https://msdn.microsoft.com/en-us/commandline/wsl/about), or use an alternate method to download the script into the vendor directory.

```shell
curl -o vendor/gmaps.js "https://maps.googleapis.com/maps/api/js?v=3.22"
```

Once in the vendor directory, the script can be built into the app. We just need to tell Ember CLI to import it using our build file:

<pre><code class="ember-cli-build.js{+22}">/*jshint node:true*/
/* global require, module */
let EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function(defaults) {
  let app = new EmberApp(defaults, {
    // Add options here
  });

  // Use `app.import` to add additional libraries to the generated
  // output files.
  //
  // If you need to use different assets in different
  // environments, specify an object as the first parameter. That
  // object's keys should be the environment name and the values
  // should be the asset to use in that environment.
  //
  // If the library that you are including contains AMD or ES6
  // modules that you would like to import into your application
  // please specify an object with the list of modules as keys
  // along with the exports of each module as its value.
  app.import('vendor/gmaps.js');

  return app.toTree();
};
</code></pre>

### Accessing the Google Maps API with a Utility

Ember utilities are reusable code that can be accessed from various parts of the application. For Super Rentals, we'll use a utility to access the Google Maps API. The utility will abstract the Google API away from our Maps service, which will allow for future reuse of the maps API within the application, easier refactoring to alternate maps implementations, and easier testing of code that depends on it.

Now that we have the maps API available to the application, we can create our map utility. Utility files can be generated using Ember CLI.

```shell
ember g util google-maps
```

The CLI `generate util` command will create a utility file and a unit test. We'll delete the unit test since we don't want to test Google code.

Our app needs a single function, `createMap`, which makes use of `google.maps.Map` to create our map element, `google.maps.Geocoder` to lookup the coordinates of our location, and `google.maps.Marker` to pin our map based on the resolved location.

<pre><code class="app/utils/google-maps.js">import Ember from 'ember';

const google = window.google;

export default Ember.Object.extend({

  init() {
    this.set('geocoder', new google.maps.Geocoder());
  },

  createMap(element, location) {
    let map = new google.maps.Map(element, { scrollwheel: false, zoom: 10 });
    this.pinLocation(location, map);
    return map;
  },

  pinLocation(location, map) {
    this.get('geocoder').geocode({address: location}, (result, status) =&gt; {
      if (status === google.maps.GeocoderStatus.OK) {
        let geometry = result[0].geometry.location;
        let position = { lat: geometry.lat(), lng: geometry.lng() };
        map.setCenter(position);
        new google.maps.Marker({ position, map, title: location });
      }
    });
  }

});
</code></pre>

### Fetching Maps With a Service

Now that we are able to generate a map element, we will implement a maps service that will keep a reference to the Map object we create, and attach the map to an element in our application

Accessing our maps API through a [service](../../applications/services) will give us several benefits

* It is injected with a [service locator](https://en.wikipedia.org/wiki/Service_locator_pattern), meaning it will abstract the maps API from the code that uses it, allowing for easier refactoring and maintenance.
* It is lazy-loaded, meaning it won't be initialized until it is called the first time. In some cases this can reduce your app's processor load and memory consumption.
* It is a singleton, which means there is only one instance of the service object in browser. This will allow us to keep map data while the user navigates around the app, so that returning to a page doesn't require it to reload its maps.

Let's get started creating our service by generating it through Ember CLI, which will create the service file, as well as a unit test for it.

```shell
ember g service maps
```

Now implement the service as follows. Note that we check if a map already exists for the given location and use that one, otherwise we call a Google Maps utility to create one.

```app/services/maps.js
import Ember from 'ember';
import MapUtil from '../utils/google-maps';

export default Ember.Service.extend({

  init() {
    if (!this.get('cachedMaps')) {
      this.set('cachedMaps', Ember.Object.create());
    }
    if (!this.get('mapUtil')) {
      this.set('mapUtil', MapUtil.create());
    }
  },

  getMapElement(location) {
    let camelizedLocation = location.camelize();
    let element = this.get(`cachedMaps.${camelizedLocation}`);
    if (!element) {
      element = this.createMapElement();
      this.get('mapUtil').createMap(element, location);
      this.set(`cachedMaps.${camelizedLocation}`, element);
    }
    return element;
  },

  createMapElement() {
    let element = document.createElement('div');
    element.className = 'map';
    return element;
  }

});
```

### Display Maps With a Component

With a service and utility that render a map to a web page element, we'll connect it to our application using a component.

Generate the map component using Ember CLI.

```shell
ember g component location-map
```

Running this command generates three files: a component JavaScript file, a template, and a test file.

Let's start by adding a `div` element to the component template. This `div` will act as a place for the 3rd party map API to render the map to.

<pre><code class="app/templates/components/location-map.hbs">&lt;div class="map-container"&gt;&lt;/div&gt;
</code></pre>

Next, update the component to append the map output to the `div` element we created.

We provide the maps service into our component by initializing a property of our component, called `maps`. Services are commonly made available in components and other Ember objects by ["service injection"](../../applications/services/#toc_accessing-services). When you initialize a property with `Ember.inject.service()`, Ember tries to set that property with a service matching its name.

With our `maps` service, our component will call the `getMapElement` function with the provided location. We append the map element we get back from the service by implementing `didInsertElement`, which is a [component lifecycle hook](../../components/the-component-lifecycle/#toc_integrating-with-third-party-libraries-with-code-didinsertelement-code). This function runs during the component render, after the component's markup gets inserted into the page.

<pre><code class="app/components/location-map.js">import Ember from 'ember';

export default Ember.Component.extend({
  maps: Ember.inject.service(),

  didInsertElement() {
    this._super(...arguments);
    let location = this.get('location');
    let mapElement = this.get('maps').getMapElement(location);
    this.$('.map-container').append(mapElement);
  }
});
</code></pre>

You may have noticed that this.get('location') refers to a property location we haven't defined. This property will be passed in to the component by its parent template below.

Finally open the template file for our `rental-listing` component and add the new `location-map` component.

<pre><code class="app/templates/components/rental-listing.hbs{+19}">&lt;article class="listing"&gt;
  &lt;a {{action 'toggleImageSize'}} class="image {{if isWide "wide"}}"&gt;
    &lt;img src="{{rental.image}}" alt=""&gt;
    &lt;small&gt;View Larger&lt;/small&gt;
  &lt;/a&gt;
  &lt;h3&gt;{{rental.title}}&lt;/h3&gt;
  &lt;div class="detail owner"&gt;
    &lt;span&gt;Owner:&lt;/span&gt; {{rental.owner}}
  &lt;/div&gt;
  &lt;div class="detail type"&gt;
    &lt;span&gt;Type:&lt;/span&gt; {{rental-property-type rental.type}} - {{rental.type}}
  &lt;/div&gt;
  &lt;div class="detail location"&gt;
    &lt;span&gt;Location:&lt;/span&gt; {{rental.city}}
  &lt;/div&gt;
  &lt;div class="detail bedrooms"&gt;
    &lt;span&gt;Number of bedrooms:&lt;/span&gt; {{rental.bedrooms}}
  &lt;/div&gt;
  {{location-map location=rental.city}}
&lt;/article&gt;
</code></pre>

After starting the server we should now see some end to end maps functionality show up on our front page!

![super rentals homepage with maps](../../images/service/style-super-rentals-maps.png)

You may now either move onto the [next feature](../subroutes/), or continue here to test the maps feature we just added.

### Unit testing a Service

We'll use a unit test to validate the service. Unit tests are more isolated than integration tests and acceptance test, and are intended for testing specific logic within a class.

For our service unit test, we'll want to verify that locations that have been previously loaded are fetched from cache, while new locations are created using the utility. We will isolate our tests from actually calling Google Maps by stubbing our map utility. On line 6 of `maps-test.js` below we create an Ember object to simulate the behavior of the utility, but instead of creating a google map, we return an empty JavaScript object.

Unit tests use the function called `this.subject` to instantiate the object to test, and lets the test pass in initial values as arguments. In our case we are passing in our fake map utility object in the first test, and passing a cache object for the second test.

<pre><code class="tests/unit/services/maps-test.js">import { moduleFor, test } from 'ember-qunit';
import Ember from 'ember';

const DUMMY_ELEMENT = {};

let MapUtilStub = Ember.Object.extend({
  createMap(element, location) {
    this.assert.ok(element, 'createMap called with element');
    this.assert.ok(location, 'createMap called with location');
    return DUMMY_ELEMENT;
  }
});

moduleFor('service:maps', 'Unit | Service | maps');

test('should create a new map if one isnt cached for location', function (assert) {
  assert.expect(4);
  let stubMapUtil = MapUtilStub.create({ assert });
  let mapService = this.subject({ mapUtil: stubMapUtil });
  let element = mapService.getMapElement('San Francisco');
  assert.ok(element, 'element exists');
  assert.equal(element.className, 'map', 'element has class name of map');
});

test('should use existing map if one is cached for location', function (assert) {
  assert.expect(1);
  let stubCachedMaps = Ember.Object.create({
    sanFrancisco: DUMMY_ELEMENT
  });
  let mapService = this.subject({ cachedMaps: stubCachedMaps });
  let element = mapService.getMapElement('San Francisco');
  assert.equal(element, DUMMY_ELEMENT, 'element fetched from cache');
});
</code></pre>

When the service calls `createMap` on our fake utility, we will run asserts to validate that it is called. In our first test notice that we expect four asserts to be run in line 17. Two of the asserts run in the test function, while the other two are run when `createMap` is called.

In the second test, only one assert is expected (line 26), since the map element is fetched from cache and does not use the utility.

Also, note that the second test uses a dummy object as the returned map element (defined on line 4). Our map element can be substituted with any object because we are only asserting that the cache has been accessed (see line 34).

The location in the cache has been [`camelized`](http://emberjs.com/api/classes/Ember.String.html#method_camelize) (line 30), so that it may be used as a key to look up our element. This matches the behavior in `getMapElement` when city has not yet been cached.

### Integration Testing the Map Component

Now lets test that the map component is relying on our service to provide map elements.

To limit the test to validating only its own behavior and not the service, we'll take advantage of the registration API to register a stub maps service. That way when Ember injects the map service into the component, it uses our fake service instead of the real one.

A stub stands in place of the real object in your application and simulates its behavior. In the stub service, define a method that will fetch the map based on location, called `getMapElement`.

<pre><code class="tests/integration/components/location-map-test.js">import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';
import Ember from 'ember';

let StubMapsService = Ember.Service.extend({
  getMapElement(location) {
    this.set('calledWithLocation', location);
    // We create a div here to simulate our maps service,
    // which will create and then cache the map element
    return document.createElement('div');
  }
});

moduleForComponent('location-map', 'Integration | Component | location map', {
  integration: true,
  beforeEach() {
    this.register('service:maps', StubMapsService);
    this.inject.service('maps', { as: 'mapsService' });
  }
});

test('should append map element to container element', function(assert) {
  this.set('myLocation', 'New York');
  this.render(hbs`{{location-map location=myLocation}}`);
  assert.equal(this.$('.map-container').children().length, 1, 'the map element should be put onscreen');
  assert.equal(this.get('mapsService.calledWithLocation'), 'New York', 'a map of New York should be requested');
});
</code></pre>

In the `beforeEach` function that runs before each test, we use the built-in function `this.register` to [register](../../applications/dependency-injection/#toc_factory-registrations) our stub service in place of the maps service. Registration makes an object available to your Ember application for things like loading components from templates and injecting services in this case.

The call to the function `this.inject.service` [injects](../../applications/dependency-injection/#toc_ad-hoc-injections) the service we just registered into the context of the tests, so each test may access it through `this.get('mapsService')`. In the example we assert that `calledWithLocation` in our stub is set to the location we passed to the component.

### Stubbing Services in Acceptance Tests

Finally, we want to update our acceptance tests to account for our new service. While it would be great to verify that a map is displaying, we don't want to hammer the Google Maps API every time we run our acceptance test. For this tutorial we'll rely on our component integration tests to ensure that the map DOM is being attached to our screen. To avoid hitting our Maps request limit, we'll stub out our Maps service in our acceptance tests.

Often, services connect to third party APIs that are not desirable to include in automated tests. To stub these services we simply have to register a stub service that implements the same API, but does not have the dependencies that are problematic for the test suite.

Add the following code after the imports to our acceptance test:

<pre><code class="/tests/acceptance/list-rentals-test.js{+3,+5,+6,+7,+8,+9,+10,-11,+12,+13,+14,+15,+16,+17}">import { test } from 'qunit';
import moduleForAcceptance from 'super-rentals/tests/helpers/module-for-acceptance';
import Ember from 'ember';

let StubMapsService = Ember.Service.extend({
  getMapElement() {
    return document.createElement('div');
  }
});

moduleForAcceptance('Acceptance | list-rentals');
moduleForAcceptance('Acceptance | list rentals', {
  beforeEach() {
    this.application.register('service:stubMaps', StubMapsService);
    this.application.inject('component:location-map', 'maps', 'service:stubMaps');
  }
});
</code></pre>

What's happening here is we are adding our own stub maps service that simply creates an empty div. Then we are putting it in Ember's [registry](../../applications/dependency-injection#toc_factory-registrations), and injecting it into the `location-map` component that uses it. That way every time that component is created, our stub map service gets injected over the Google maps service. Now when we run our acceptance tests, you'll notice that maps do not get rendered as the test runs.

![acceptance tests without maps](../../images/service/acceptance-without-maps.png)