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
  $c: dy-deployable()
```

--- 

Deployables allow you to define a component with internal state properties, which you can then reference from the content of the component. Then you can define different versions and transformations of the component, with different values of those state properties, and the changes you make will cascade to all the references.

Here's a basic example.

```sass
.deployable-component
  // activates this selector as a deployable. only valid on a single simple selector
  $component: dy-deployable()
  // creates a new internal property, and sets the default to red
  $component: dy-define-state($component, color, red)

  // normal css attributes are of course fine
  width: 100%

  // uses the color internal property, outputting its value to border-color 
  $component: dy-output($component, border-color, color)

  &.primary
    // creates a new version, with blue as its state
    $component: dy-define-version($component, color, blue)

  &:hover
    // creates a new transform,
    // saying that every version should have a lightened hover state
    $component: dy-define-transform($component, lighten, 'this.color', 10%)

  // builds the deployable
  +dy-build($component)
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
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  width: 100%

  $c: dy-output($c, color)
  .nested-block
    $c: dy-output($c, border-color, color)
    $c: dy-output-function($c, background-color, adjust-hue, 'this.color', 180)

  &.primary
    $c: dy-define-version($c, color, blue)
    
  &.success
    $c: dy-define-version($c, color, green)

  &:hover
    $c: dy-define-transform($c, lighten, 'this.color', 10%)

  +dy-build($c)
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
- `@function dy-define-state($c, $name, $value)`
- `@function dy-define-version($c, $name, $value)`
- `@function dy-define-transform($c, $name, $func, $func-args...)`
- `@function dy-output($c, $css-prop, $var-name: null)`
- `@function dy-output-function($c, $css-prop, $func, $func-args...)`
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
  $c: dy-deployable()
  // everything at the base of the component is the "default" version
  $c: dy-define-state($c, color, gray)
```

Now that variable can be referred to in the content, given different values in different versions, and changed in transforms.

**Note:** you can't call `dy-define-state` in versions, transforms, or nested blocks.

```sass
.deployable-component
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  .nested-block
    // -> ERROR, invalid location to set deployable state
    $c: dy-define-state($c, color, green)
    $c: dy-output($c, color)

  &.primary
    // -> ERROR, can't call dy-define-state in version
    // use dy-define-version instead
    $c: dy-define-state($c, color, red)
```


## Content References

In order to output the state values in a way that will be remembered and used for every Version, you can use the `dy-output` and `dy-output-function` functions. You can even call these functions in nested blocks, and they'll still work exactly as expected.

```sass
.deployable-component
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  // has two forms
  // one that outputs to the css property with the same name
  $c: dy-output($c, color)
  // -> color: the value of color

  // and one that outputs to a named css property
  $c: dy-output($c, border-color, color)
  // -> border-color: the value of color


  // supply the css property to be output to,
  // the function name,
  // and the list of arguments,
  // using the format 'this.statevar' to refer to state properties
  $c: dy-output-function($c, background-color, adjust-hue, 'this.color', 180)


  .nested-block
    // every version will have a .nested-block with the appropriate background-color
    $c: dy-output($c, background-color, color)


  // ... versions and transforms ...

  +dy-build($c)
```


### Content Overrides

In different Versions and Transforms, you can override the content outputs, or add new ones.

```sass
.deployable-component
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)
  $c: dy-output($c, border-color, color)

  &.primary
    $c: dy-define-state($c, color, green)

    // this will override the normal "border-color: color" output
    $c: dy-output-function($c, border-color, lighten, 'this.color', 10%)

    // this will only output for this version
    $c: dy-output($c, background-color, color)
    

  +dy-build($c)
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
  $c: dy-deployable()

  // the default value
  $c: dy-define-state($c, color, black)

  $c: dy-output($c, color)
  $c: dy-output($c, border-color, color)
  $c: dy-output-function($c, background-color, lighten, 'this.color', 90%)

  &.primary
    $c: dy-define-version($c, color, blue)
    
  &.success
    $c: dy-define-version($c, color, green)

  &.danger
    $c: dy-define-version($c, color, red)

  +dy-build($c)
```

Each of those types (`.primary`, `.success`, `.danger`) can't coexist with the others. A button can't be both `.success` and `.danger`. That's what makes them Versions, their separateness.


### Interleaving Versions

Even though Versions shouldn't overlap, you can have different *kinds* of Versions changing different qualities. As long as they don't conflict with each other everything will work. A simple example is a button with different Versions for colors, and another set of Versions for sizes.

```sass
.button
  $c: dy-deployable()

  // states, outputs, and versions for coloring
  $c: dy-define-state($c, color, black)
  $c: dy-output($c, color)
  $c: dy-output($c, border-color, color)

  &.primary
    $c: dy-define-version($c, color, blue)
    
  &.success
    $c: dy-define-version($c, color, green)


  // states, outputs, and versions for sizing
  $c: dy-define-state($c, font-size, 16px)
  $c: dy-output($c, font-size)
  $c: dy-output($c, padding, font-size)

  &.big
    $c: dy-define-version($c, font-size, 20px)

  &.small
    $c: dy-define-version($c, font-size, 12px)

  +dy-build($c)
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
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)

  &.open:first-child
    $c: dy-define-version($c, color, silver)

  &.open
    $c: dy-define-version($c, color, blue)

  &:first-child
    $c: dy-define-version($c, color, green)

  +dy-build($c)
``` -->


## Transforms

You can define Transforms on the component. These work very similarly to Versions, but instead of setting the state directly, you define a change to be made to it.

```sass
.deployable-component
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)

  &.success
    $c: dy-define-version($c, color, green)

  &:hover
    $c: dy-define-transform(color, fade-out, 'this.color', 0.2)

  +dy-build($c)
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
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)

  &:hover
    $c: dy-define-transform($c, color, fade-out, 'this.color', 0.2)

  &.open
    $c: dy-define-transform($c, color, adjust-hue, 'this.color', 180deg)

  +dy-build($c)
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
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)

  &:hover
    $c: dy-define-transform($c, color, fade-out, 'this.color', 0.2)
    $c: dy-output-function($c, color, desaturate, 'this.color', 10%))

  &.open
    $c: dy-define-transform($c, color, adjust-hue, 'this.color', 180deg)
    $c: dy-output-function($c, color, desaturate, 'this.color', 20%))

  +dy-build($c)
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
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)

  &:hover
    $c: dy-define-transform($c, color, fade-out, 'this.color', 0.2)
    $c: dy-output-function($c, color, desaturate, 'this.color', 10%))

  &.open
    $c: dy-define-transform($c, color, adjust-hue, 'this.color', 180deg)
    $c: dy-output-function($c, color, desaturate, 'this.color', 20%))

  &.open:hover
    $c: dy-define-transform($c, color, adjust-hue, 'this.color', 180deg)

  +dy-build($c)
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
  $c: dy-deployable()
  $c: dy-define-state($c, color, red)

  $c: dy-output($c, color)

  &.success
    $c: dy-define-version($c, color, green)

    // -> ERROR, can't mingle versions and transforms
    $c: dy-define-transform(color, saturate, 'this.color', 10%)

  &:hover
    $c: dy-define-transform(color, fade-out, 'this.color', 0.2)
    
    // -> ERROR, can't mingle versions and transforms
    $c: dy-define-version($c, color, gray)

  +dy-build($c)
```


## Inheritance

You can create secondary Deployables that inherit the states, versions, and transforms of existing Deployables. Just make the component object available to other scopes, and pass it to `dy-deployable`. 

In order to also get the benefits of Sass `@extend`, call the `dy-extend` mixin. Fixing the inconvenience of having to call two different things in order to extend a component is one of the features planned for the next version.

```sass
.parent-component
  $c: dy-deployable()
  $c: dy-define-state($c, color, black)

  // here's a normal css attribute
  // which will be carried over by dy-extend
  width: 100%

  $c: dy-output($c, color)

  &:hover
    $c: dy-define-transform(color, fade-out, 'this.color', 0.2)

  +dy-build($c)

  // this is necessary to make the $c available to other scopes
  $parent: $c !global

.child-component
  $c: dy-deployable($parent)
  +dy-extend($parent)

  &.primary
    $c: dy-define-version($c, color, blue)

  &.success
    $c: dy-define-version($c, color, green)

  +dy-build($c)
```

This is just as useful with extend-only selectors. Just skip the `dy-build` call.

```sass
%hoverable
  $c: dy-deployable()

  &:hover
    $c: dy-define-transform(color, fade-out, 'this.color', 0.2)

  $hoverable: $c !global

.deployable-component
  $c: dy-deployable($hoverable)
  +dy-extend($hoverable)

  // ... etc ...

  +dy-build($c)
```

### Extendable Libraries

Since Deployables are essentially just maps stored in a variable like `$c`, they can be shared in libraries that would allow users to extend existing components. As long as the variables are made global and available, anyone can work with them.

```sass
// in a package called 'cool-buttons'
// 'main.sass'
%cool-btn
  $c: dy-deployable()

  $c: dy-define-state($c, color, black)
  $c: dy-output($c, color)
  $c: dy-output($c, border-color, color)

  &.primary
    $c: dy-define-version($c, color, blue)
    
  &.success
    $c: dy-define-version($c, color, green)

  $cool-btn-component: $c !global


// in user code
@import 'cool-buttons/main'
.extended-btn
  $c: dy-deployable($cool-btn-component)
  +dy-extend($cool-btn-component)

  // overriding an old Version
  &.primary
    $c: dy-define-version($c, color, cyan)

  // creating a new Version
  &.danger
    $c: dy-define-version($c, color, red)

  +dy-build($c)
```


## Future Plans

**Features**

- [x] transform compounding
- [ ] hidden `$c` object saving, so the calls aren't so cluttered
- [ ] getting rid of `dy-extend` and calling `@extend` in `dy-deployable`
- [ ] computed states
- [ ] more flexible version definitions, allowing the `&` to be in different places
- [ ] "related" versions, ones that are derived from a base value

**Administrivia**

- [ ] robust testing
- [ ] thorough errors and warnings


## Contributing

If you'd like to contribute, just fork the project and then make a pull request!








<!-- ```sass
.component
  +dy-deployable

  +dy-define-state(margin, '1ru')

  +dy-define-computed-state(margin-left, eval-margin, 'left', 'this.margin')

  // sass doesn't output if the thing you output is null
  // if color is null, nothing will be output
  +dy-output(color)

  // if the function you pass could return null, then nothing will be output
  +dy-output-function(color, thing-that-might-null, 'this.color')
``` -->
