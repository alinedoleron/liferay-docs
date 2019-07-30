---
header-id: rendering-field-types
---

# Rendering Field Types

[TOC levels=1-4]

Now it's time to write the front-end template necessary to render your field type.

## Writing the Soy Templates

| **Note:** [Closure templates](https://developers.google.com/closure/templates/)
| are a templating system for building UI elements. @product@ developers chose to
| build the Forms UI with closure templates because they enable a smooth,
| responsive repainting of the UI as a user enters data. With closure templates
| there's no need to reload the entire page from the server when the UI is updated
| by the user: only the relevant portion of the page is updated from the server.
| This makes for a smooth user experience.

Create

    src/main/resources/META-INF/resources/Slider.soy

and populate it with this:

    {namespace Slider}

    /**
    * Prints the slider field.
    */
    {template .render}
        {@param name: string}
        {@param label: string}
        {@param showLabel: bool}
        {@param value: ?}

        {call FieldBase.render}
            {param contentRenderer kind="html"}
                {call .content}
                    {param name: $name /}
                    {param value: $value /}
                {/call}
            {/param}
            {param name: $name /}
            {param label: $label /}
            {param showLabel: $showLabel /}
        {/call}
    {/template}

    {template .content}
        {@param name: string}
        {@param value: ?}
        {let $attributes kind="attributes"}
            class="ddm-field-slider form-control slider"

            type="range"

            min="1"

            max="100"

            id="myRange"

            name="{$name}"

            {if $value}
                value="{$value}"
            {/if}
        {/let}

        <input {$attributes}>
    {/template}

There are three important things to do in the template:

1.  Define the template namespace. The template namespace can define multiple
    templates for your field type by adding the namespace as a prefix.

        {namespace Slider}

2.  Describe the template parameters. The template above uses some of the
    parameters as flags to display or hide some parts of the HTML (for example,
    the `$showLabel` parameter). All listed parameters are available by default.

        {@param name: string}
        {@param label: string}
        {@param showLabel: bool}
        {@param value: ?}

5.  Write the template logic (everything encapsulated by the
    `{template .render}...{/template}` block). In the above example the template does
    these things:

    - Call the FieldBase template, which is the template responsible to render the base
    of all fields. This template has a parameter called ContentRenderer that receives the
    html content specific of a field type.

    - Provides the markup for the slider field in the <input> tag. In this case a text input field is defined.

Now that you have the template defined, you need to call it from a register template.

Create

    src/main/resources/META-INF/resources/SliderRegister.soy

and populate it with this:

    {namespace SliderRegister}

    {deltemplate PageRenderer.RegisterFieldType variant="'slider'"}
        {call Slider.render data="all" /}
    {/deltemplate}

Once the templates are defined, write the JavaScript file modeling your field.

## Writing the JavaScript Files

Create a `Slider.es.js` file and give it these contents:

    import '../FieldBase/FieldBase.es';
    import './SliderRegister.soy.js';
    import templates from './Slider.soy.js';
    import Component from 'metal-component';
    import Soy from 'metal-soy';
    import {Config} from 'metal-state';

    /**
     * Slider Component
     */
    class Slider extends Component {}

    Slider.STATE = {
        /**
        * @default undefined
        * @instance
        * @memberof Slider
        * @type {?(string|undefined)}
        */

        name: Config.string().required(),

        /**
        * @default undefined
        * @instance
        * @memberof Slider
        * @type {?(string|undefined)}
        */

        value: Config.string().value('')
    }

    // Register component
    Soy.register(Slider, templates);

    export default Slider;


The JavaScript above creates a component called `Slider`.

This file is entirely boilerplate. In fact, if you use Blade CLI to generate a
field type module, you won't need to touch this file. Functionally, it's a
JavaScript file that defines the dependencies of the declared JavaScript
components (`requires...`), and where the files are located (`path...`).
![Figure 1: Add your own form field types to the Forms application.](../../../images/forms-slider-field-type.png)

If you build and deploy your new field type module, you get exactly what you
described in the `Slider.soy` file: a input field of type range.  We can interact with the Slider, since this element was created to allow us to move the slider controler across the range. But we don't want to lose the value we've positined the slider when we save the Form. Now, we're going to add this behaviour to the field.

## Adding Behavior to the Field

To do more than move across the range, define an additional behavior to handle the input in the `Slider.es.js` file.

Add into Slider class, the method _handleFieldChanged:

    /**
     * Slider Component
     */
    class Slider extends Component {

        dispatchEvent(event, name, value) {
            this.emit(name, {
                fieldInstance: this,
                originalEvent: event,
                value
            });
        }

        _handleFieldChanged(event) {
            const {value} = event.target;

            this.setState(
                {
                    value
                },
                () => this.dispatchEvent(event, 'fieldEdited', value)
            );
        }
    }


Now add a the parameter _handleFieldChanged to Slider.soy file, to handle the Slider input. You'll need to add the parameter:

    {@param? _handleFieldChanged: any}

into template .render, and pass it on the call of FieldBase contentRenderer.

    {param _handleFieldChanged: $_handleFieldChanged /}

This parameter will be used on data-input event of input tag:

    data-oninput="{$_handleFieldChanged}"

The Slider.soy file now will be:

    {namespace Slider}

    /**
    * Prints the Slider field.
    */
    {template .render}
        {@param? _handleFieldChanged: any}
        {@param name: string}
        {@param label: string}
        {@param showLabel: bool}
        {@param value: ?}

        {call FieldBase.render}
            {param contentRenderer kind="html"}
                {call .content}
                    {param _handleFieldChanged: $_handleFieldChanged /}
                    {param name: $name /}
                    {param value: $value /}
                {/call}
            {/param}
            {param name: $name /}
            {param label: $label /}
            {param showLabel: $showLabel /}
        {/call}
    {/template}

    {template .content}
        {@param? _handleFieldChanged: any}
        {@param name: string}
        {@param value: ?}
        {let $attributes kind="attributes"}
            class="ddm-field-slider form-control slider"

            data-oninput="{$_handleFieldChanged}"

            id="myRange"

            min="1"

            max="100"

            name="{$name}"

            type="range"

            {if $value}
                value="{$value}"
            {/if}
        {/let}

        <input {$attributes}>
    {/template}

This method updates the field's value.

![Figure 2: Slider in action.](../../../images/slider-in-action.gif)

Now you know how to create a new field type and handle its behavior. Currently,
the field type only contains the default settings it inherits from its
superclasses. If that's not sufficient, create additional settings for your
field type. See the next tutorial to learn how.
