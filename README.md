# Proposal - named/mapped base URLs

This proposal is to add two attributes to the `<base>` tag called `name` and `map`.

## What are we trying to solve?

Essentially, we want to be able to specify a bare name in a URL (e.g. import specifiers) and have that name translated to a real URL on request.

```js
import example from 'example';
```

Translates to:

```js
import example from 'https://example.com/example.js';
```

Having said that, I won't be mentioning ES Modules for a large portion of this proposal because the specifics of ES Modules are largely irrelavant to the proposed change. 

## Constraints:

- Must work for all resources types (js, css, etc)
- Must work with a deep dependency tree
- Must be able to be locked for integrity

## Working with all resource types

Because this needs to work for all resource types I propose that we extend the `<base>` tag to provide this functionality. The base tag already performs similar translations and with a few simple modifications we can leverage that here. 

I propose that if a base tag has a `name` attribute it creates a new concept called `NamedBaseURL`.

```html
<base href="./deeplynestedpath/assets" name="assets">
```

This works a lot like the standard base tag except it only remaps relative URLs that don't begin with `./` or `../` and starts with with the name from a `NamedBaseURL`.

```html
<link rel="stylesheet" href="assets/main.css">
```

Translates to:

```html
<link rel="stylesheet" href="./deeplynestedpath/assets/main.css">
```

In practice, if you want to name a third party bundled resource you could just point to a specific resource instead of treating it like a base url.

```html
<base href="https://cdn.jsdelivr.net/npm/jquery@3.2.1/dist/jquery.min.js" name="jquery">
<script src="jquery"></script>
```

Translates to:

```html
<script src="https://cdn.jsdelivr.net/npm/jquery@3.2.1/dist/jquery.min.js"></script>
```

## Working with a deep dependency tree

Once we have the concept of a `NamedBaseURL`, I propose we add a second base tag attribute called `map`. The `map` attribute points to a json file with a key value pair of names and URLs. This in turn creates each `NamedBaseURL` accordingly. A relative URL value is calculated relative to the location of the map json file.

basemap.json

```json
{
    "assets": "./deeplynestedpath/assets",
    "jquery": "https://cdn.jsdelivr.net/npm/jquery@3.2.1/dist/jquery.min.js"
}
```

```html
<base map="basemap.json">
<link rel="stylesheet" href="assets/main.css">
<script src="jquery"></script>
```

To create a deeper dependency that will have it's own `NamedBaseURL` map we can simply use both attributes `name` and `map`. This concatentates the name with the keys from the specified map.

alpha-basemap.json

```json
{
    "bootstrap": "https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist"
}
```

```html
<base name="alpha" map="alpha-basemap.json">
<link rel="stylesheet" href="alpha/bootstrap/css/bootstrap.css">
<script src="alpha/bootstrap/js/bootstrap.js"></script>
```

Then it becomes fairly straightforward to create a deep dependency tree if we just add a special key called `@dependencies`. It holds a key value pair for additional maps which are fetched and creates new `NamedBaseURL` objects with both `name` and `map` accordingly.

bravo-basemap.json

```json
{
    "@dependencies": {
        "alpha":  "alpha-basemap.json"
    }
}
```

```html
<base name="bravo" map="bravo-basemap.json">
<link rel="stylesheet" href="bravo/alpha/bootstrap/css/bootstrap.css">
<script src="bravo/alpha/bootstrap/js/bootstrap.js"></script>
```

Could be equivalent to

```html
<base name="bravo/alpha" map="alpha-basemap.json">

<link rel="stylesheet" href="bravo/alpha/bootstrap/css/bootstrap.css">
<script src="bravo/alpha/bootstrap/js/bootstrap.js"></script>
```



For deep depencencies the URL translation would need to account for relative named base urls from the context of the resource. In other words, when `alpha` is requested from within the context of `bravo` (from inside a script or css) it should translate to `bravo/alpha` before ultimately translating to the full URL. However, I think this would be pretty straightforward and simple to understand.

## Locking for integrity

Since a dependency potentially isn't bundled or tarballed like npm we would then need a checksum/hash for each individual sub resource. It could be mapped accordingly with an `@integerity` property. Of course you would need to know all the resources for the dependency before hand. Internally, the host environment would just check if a resource has a mapped hash and throw an error if the hash does not match.

basemap.lock.json

```json
{
    "alpha": "https://example.cdn/alpha@1.0.0/dist.js",
    "bravo": "https://example.cdn/bravo@1.0.0",
    "@dependencies": {
      "charlie": "https://example.cdn/charlie@1.0.0/basemap.json"
    },
    "@integrity": {
        "alpha": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "bravo/dist.js": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "bravo/img.png": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "charlie/dist.js": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "charlie/style.css": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "charlie/delta/dist.js": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "charlie/delta/bg.png": "sha1-xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
}
```

## Example

See https://github.com/shannon/named-base-url-proposal/tree/master/example for a quick example of how it would all work together