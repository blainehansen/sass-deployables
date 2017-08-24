# sass-deployables

> A library for creating object-oriented sass components.

**Proudly sponsored by Marketdial**

<p>
<a href="http://marketdial.com">
<img src="https://cdn.rawgit.com/blainehansen/sass-deployables/master/marketdial-logo.svg" alt="MarketDial logo" title="MarketDial" width="35%">
</a>
</p>

---

```bash
npm install --save sass-deployables
```

```sass
@import 'path/to/node_modules/sass-deployables'

// for example, if using webpack node-sass
@import '~sass-deployables'

.component
  +dy-deployable

  // ...
```

--- 

Deployables allow you to define a component with internal state properties, which you can then reference from the content of the component. Then you can define different versions and transformations of the component, with different values of those state properties, and the changes you make will cascade to all the references.

Here's a basic example.

```sass
.deployable-component
  // activates this selector as a deployable. only valid on a single simple selector
  +dy-deployable
  // creates a new internal property, and sets the default to red
  +dy-define-state(color, red)

  // normal css attributes are of course fine
  width: 100%

  // uses the color internal property, outputting its value to border-color 
  +dy-output(border-color, color)

  &.primary
    // creates a new version, with blue as its state
    +dy-define-version(color, blue)

  &:hover
    // creates a new transform,
    // saying that every version should have a lightened hover state
    +dy-define-transform(lighten, 'this.color', 10%)

  // builds the deployable
  +dy-build
```

This will output something equivalent to this:

```sass
.deployable-component
  border-color: red

  width: 100%

  &:hover
    border-color: lighten(red, 20%)

  &.primary
    border-color: blue

    &:hover
      border-color: lighten(blue, 20%)
```

See how the Deployable remembered the reference (`dy-output`), and output an adjusted version of it for `.primary`? And notice how the hover transformation was only defined once but was output for both versions?

This gets much more powerful when you have more references, versions, and transforms.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  width: 100%

  +dy-output(color)
  .nested-block
    +dy-output(border-color, color)
    +dy-output-function(background-color, adjust-hue, 'this.color', 180)

  &.primary
    +dy-define-version(color, blue)
    
  &.success
    +dy-define-version(color, green)

  &:hover
    +dy-define-transform(lighten, 'this.color', 10%)

  +dy-build
```
Compiles to something like:

```sass
.deployable-component
  color: red
  width: 100%

  .nested-block
    border-color: red
    background-color: adjust-hue(red, 180)

  &:hover
    color: lighten(red, 20%)

    .nested-block
      border-color: lighten(red, 20%)
      background-color: adjust-hue(lighten(red, 20%), 180)

  &.primary
    color: blue

    .nested-block
      border-color: blue
      background-color: adjust-hue(blue, 180)

    &:hover
      color: lighten(blue, 20%)

      .nested-block
        border-color: lighten(blue, 20%)
        background-color: adjust-hue(lighten(blue, 20%), 180)

  &.success
    color: green

    .nested-block
      border-color: green
      background-color: adjust-hue(green, 180)

    &:hover
      color: lighten(green, 20%)

      .nested-block
        border-color: lighten(green, 20%)
        background-color: adjust-hue(lighten(green, 20%), 180)
```

Using Deployables will help you cut down on a huge amount of redundancy.

## API

There are six functions to create Deployables, and one mixin to build them.

- `@function dy-deployable($parent-component: null)`
- `@function dy-define-state($name, $value)`
- `@function dy-define-version($name, $value)`
- `@function dy-define-transform($name, $func, $func-args...)`
- `@function dy-output($css-prop, $var-name: null)`
- `@function dy-output-function($css-prop, $func, $func-args...)`
- `@mixin dy-build($c)`

Thorough explanations are given below.

## Concepts

Deployables are made of a few core concepts:

- States. Internal variables the component has.
- Versions. Distinct types of the component. Each version should have at least one state variable that is different than all the others.
- Transforms. Changes that should be applied to the state variables no matter what they are.
- References, or the Content. The uses of the state variables, made in the content of the component.


## States

Defining a new state variable is simple. Just call `dy-define-state` in the base of the component, in the "default" version.

```sass
.deployable
  // initialize
  +dy-deployable
  // everything at the base of the component is the "default" version
  +dy-define-state(color, gray)
```

Now that variable can be referred to in the content, given different values in different versions, and changed in transforms.

**Note:** you can't call `dy-define-state` in versions, transforms, or nested blocks.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  .nested-block
    // -> ERROR, invalid location to set deployable state
    +dy-define-state(color, green)
    +dy-output(color)

  &.primary
    // -> ERROR, can't call dy-define-state in version
    // use dy-define-version instead
    +dy-define-state(color, red)
```


## Content References

In order to output the state values in a way that will be remembered and used for every Version, you can use the `dy-output` and `dy-output-function` functions. You can even call these functions in nested blocks, and they'll still work exactly as expected.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  // has two forms
  // one that outputs to the css property with the same name
  +dy-output(color)
  // -> color: the value of color

  // and one that outputs to a named css property
  +dy-output(border-color, color)
  // -> border-color: the value of color


  // supply the css property to be output to,
  // the function name,
  // and the list of arguments,
  // using the format 'this.statevar' to refer to state properties
  +dy-output-function(background-color, adjust-hue, 'this.color', 180)


  .nested-block
    // every version will have a .nested-block with the appropriate background-color
    +dy-output(background-color, color)


  // ... versions and transforms ...

  +dy-build
```


### Content Overrides

In different Versions and Transforms, you can override the content outputs, or add new ones.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)
  +dy-output(border-color, color)

  &.primary
    +dy-define-state(color, green)

    // this will override the normal "border-color: color" output
    +dy-output-function(border-color, lighten, 'this.color', 10%)

    // this will only output for this version
    +dy-output(background-color, color)
    

  +dy-build
```
Compiles to something like:

```sass
.deployable-component
  color: red
  border-color: red

  &.primary
    color: green
    border-color: lighten(green, 10%)
    background-color: green
```



## Versions

A Version is any different kind of the component that is distinct from all the others, and doesn't overlap with them.

A common example is different kinds of buttons:

```sass
.button
  +dy-deployable

  // the default value
  +dy-define-state(color, black)

  +dy-output(color)
  +dy-output(border-color, color)
  +dy-output-function(background-color, lighten, 'this.color', 90%)

  &.primary
    +dy-define-version(color, blue)
    
  &.success
    +dy-define-version(color, green)

  &.danger
    +dy-define-version(color, red)

  +dy-build
```

Each of those types (`.primary`, `.success`, `.danger`) can't coexist with the others. A button can't be both `.success` and `.danger`. That's what makes them Versions, their separateness.


### Interleaving Versions

Even though Versions shouldn't overlap, you can have different *kinds* of Versions changing different qualities. As long as they don't conflict with each other everything will work. A simple example is a button with different Versions for colors, and another set of Versions for sizes.

```sass
.button
  +dy-deployable

  // states, outputs, and versions for coloring
  +dy-define-state(color, black)
  +dy-output(color)
  +dy-output(border-color, color)

  &.primary
    +dy-define-version(color, blue)
    
  &.success
    +dy-define-version(color, green)


  // states, outputs, and versions for sizing
  +dy-define-state(font-size, 16px)
  +dy-output(font-size)
  +dy-output(padding, font-size)

  &.big
    +dy-define-version(font-size, 20px)

  &.small
    +dy-define-version(font-size, 12px)

  +dy-build
```

Now when you make buttons in your templates, you can use Versions for colors and for sizes together, like `.button.primary.big`. As long as these interleaved Versions don't operate on the same States, they'll work great.


### Version Selector Limitations

Right now, the rules for versions are pretty specific. A valid version is any selector that starts with `&`. Here are some examples of valid versions.

- Compounding selectors: `&#id`, `&.big`
- Pseudo-elements: `&::before`
- Attributes or psueudo-classes: `&:active`, `&[target=_blank]`
- Suffixes: `&-suffix`


<!-- ### Output Order

Deployables can't guarantee that 

Although it's not a good idea to have versions that could potentially conflict with each other, it's technically allowed by this library. 

no guarantee of output order.


Even though 

When different versions conflict with each other, the one defined most recently wins. So in the above example, if the element was both `.open` and `:first-child`, it would be green since `:first-child` was defined last.

You can create more specific versions to handle these kinds of situations.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)

  &.open:first-child
    +dy-define-version(color, silver)

  &.open
    +dy-define-version(color, blue)

  &:first-child
    +dy-define-version(color, green)

  +dy-build
``` -->


## Transforms

You can define Transforms on the component. These work very similarly to Versions, but instead of setting the state directly, you define a change to be made to it.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)

  &.success
    +dy-define-version(color, green)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)

  +dy-build
```
```sass
.deployable-component
  color: red

  &:hover
    color: fade-out(red, 0.2)

  &.success
    color: green

    &:hover
      color: fade-out(green, 0.2)
```

Transforms will be applied to all versions, so you only have to declare them once.


### Compounding Transforms

All transforms you define will be compounded on top of each other, in every combination. This means that when something has multiple transforms applied to it at the same time, all the transform functions will be run in order of their definition.

Here's an example.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)

  &.open
    +dy-define-transform(color, adjust-hue, 'this.color', 180deg)

  +dy-build
```
```sass
.deployable-component
  color: red

  // first both of them by themselves
  &:hover
    color: fade-out(red, 0.2)

  &.open
    color: adjust-hue(red, 180deg)

  // then both together
  &.open:hover
    // first the fade-out is applied,
    // since :hover was specified first,
    // and then the adjust-hue
    color: adjust-hue(fade-out(red, 0.2), 180deg)
```

This compounding also occurs for content references. When two transforms both specify an output for an attribute, the one that was defined *last* will win.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)
    +dy-output-function(color, desaturate, 'this.color', 10%))

  &.open
    +dy-define-transform(color, adjust-hue, 'this.color', 180deg)
    +dy-output-function(color, desaturate, 'this.color', 20%))

  +dy-build
```
```sass
.deployable-component
  color: red

  // for all of these, the transform function(s) happen before the output function
  &:hover
    color: desaturate(fade-out(red, 0.2), 10%)

  &.open
    color: desaturate(adjust-hue(red, 180deg), 20%)

  &.open:hover
    // here 20% desaturate happens, because open was defined last
    color: desaturate(adjust-hue(fade-out(red, 0.2), 180deg), 20%)
```

If you like, you can provide your own compound Transforms. These will override all transform functions or content references you defined in the constituent Transforms, and only use the things you explicitly define.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)
    +dy-output-function(color, desaturate, 'this.color', 10%))

  &.open
    +dy-define-transform(color, adjust-hue, 'this.color', 180deg)
    +dy-output-function(color, desaturate, 'this.color', 20%))

  &.open:hover
    +dy-define-transform(color, adjust-hue, 'this.color', 180deg)

  +dy-build
```
```sass
.deployable-component
  color: red

  &:hover
    color: desaturate(fade-out(red, 0.2), 10%)

  &.open
    color: desaturate(adjust-hue(red, 180deg), 20%)

  &.open:hover
    // notice how this doesn't use anything from :hover and .open?
    // neither the fade-out from :hover, nor the desaturate from either
    // this transform stands on it's own
    color: adjust-hue(red, 180deg)
```

Obviously, the more Transforms you have the crazier the number of combinations can get (`(2 ^ n) - 1` to be exact). For the next version of this functionality, there will be an ability to control which Transforms compound and which don't, but for now just don't go too crazy.



## Mingling Versions and Transforms

It isn't allowed to mix versions and transforms in the same block.

```sass
.deployable-component
  +dy-deployable
  +dy-define-state(color, red)

  +dy-output(color)

  &.success
    +dy-define-version(color, green)

    // -> ERROR, can't mingle versions and transforms
    +dy-define-transform(color, saturate, 'this.color', 10%)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)
    
    // -> ERROR, can't mingle versions and transforms
    +dy-define-version(color, gray)

  +dy-build
```


## Inheritance

You can create secondary Deployables that inherit the states, versions, and transforms of existing Deployables. Just make the component object available to other scopes, and pass it to `dy-deployable`. `@extend` is called for you when you do this.

```sass
.parent-component
  +dy-deployable
  +dy-define-state(color, black)

  // here's a normal css attribute
  // which will be extended when you inherit
  width: 100%

  +dy-output(color)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)

  +dy-build


.child-component
  // calls @extend for you under the hood
  +dy-deployable('.parent-component')

  &.primary
    +dy-define-version(color, blue)

  &.success
    +dy-define-version(color, green)

  +dy-build
```

This is just as useful with extend-only selectors. Just skip the `dy-build` call.

```sass
%hoverable
  +dy-deployable

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 0.2)


.deployable-component
  +dy-deployable('%hoverable')

  // ... etc ...

  +dy-build
```

Because they can be inherited, Deployables can be shared in libraries.

```sass
// in a package called 'cool-buttons'
// 'main.sass'
%cool-btn
  +dy-deployable

  +dy-define-state(color, black)
  +dy-output(color)
  +dy-output(border-color, color)

  &.primary
    +dy-define-version(color, blue)
    
  &.success
    +dy-define-version(color, green)

  &:hover
    +dy-define-transform(color, fade-out, 'this.color', 10%)


// in user code
@import 'cool-buttons/main'
.extended-btn
  +dy-deployable('%cool-btn')

  // overriding an old Version
  &.primary
    +dy-define-version(color, cyan)

  // creating a new Version
  &.danger
    +dy-define-version(color, red)

  // creating a new Transform
  &:active
    +dy-define-transform(color, saturate, 'this.color', 10%)

  // ... etc ...

  +dy-build
```


## Roadmap

**Features**

- [x] transform compounding
- [x] hidden `$c` object saving, so the calls aren't so cluttered
- [x] getting rid of `dy-extend` and calling `@extend` in `dy-deployable`
- [ ] control over compounding
- [ ] ability to strip versions and transforms out of extend deployables
- [ ] more flexible version definitions, allowing the `&` to be in different places
- [ ] "related" versions, ones that are derived from a base value

**Administrivia**

- [ ] robust testing with [sassaby](https://github.com/ryanbahniuk/sassaby)
- [ ] thorough errors and warnings


## Contributing

If you'd like to contribute, just fork the project and then make a pull request!

In the `dev` directory, there is a `_dev.sass` file that is set up for you to play around with the library. Running `npm run dev` in the terminal will run that file, and compile to `dev/output.css`.