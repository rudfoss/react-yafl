# React-YAFL

Yet Another Form Library for React is a form-logic control library inspired by the great [react-hook-form](https://react-hook-form.com/) library.

## Motivation

[React-hook-form](https://react-hook-form.com/) is a great and relatively simple library. It is relatively simple to get started with and works exceptionally well with pure native components. But after using it with component libraries of different types including [Material-UI](https://material-ui.com/) and [Fluent-UI](https://developer.microsoft.com/en-us/fluentui#/) I've been stumped multiple times by the extraneous workarounds needed to register fields as well as supporting proper validation and error handling. To that end I decided to try my own hand at creating a simple form management library with the goal of having a simple, generic and consistent API that can be used both with primitive input elements as well as custom component libraries.

## Features

- A generic store for field values for a form.
- Validation entry points for validating values in isolation as well as alongside the other values in the form (custom validation logic is required).
- A clean and simple, strongly typed API (TypeScript).
- Easily connect with other state management libraries such as Redux.
- Sightly opinionated types and interfaces.

## Core concepts

To use this library you should quickly read up on a few core concepts.

### Store

At all times the live values of the form is kept in a store. Theses values are indexed by a field name that commonly maps to the name of the input component. Fields may be added to or removed from the store dynamically at any time.

### Field

A field is an object that has a unique name and a value. The value can be of any type including complex objects and arrays. In addition to this the libray can also keep track of:

- Whether the field has been modified (is dirty)
- Whether it has been visited
- Its current validation state

## Form API

The library core does not automatically collect any information (though you may use helper functions to accomplish this). In order to get full functionality of the form you need to connect some props to your input components. These props are normalized in order to be consistend and offer the widest possible range of compatiblity. However you may need to provide intermediate functions that translate into types that the library understands.

Any form starts with a component using the hook `useForm` along with the context provider.

```tsx
import React from "react"
import { useForm, FormContext, handleSubmit } from "react-yafl"
import { submitForm, validateForm } from "./myFormLibrary" // You provide this part

export const MyForm = () => {
  const form = useForm({
    onSubmit: submitForm, // This function is memoized internally for performance reasons so it should be pure
    onValidate: validateForm, // This function is memoized internally for performance reasons so it should be pure
  })

  return (
    <FormContext form={form}>
      <form onSubmit={handleSubmit(form)}>
        <MyyFormTextInput name="first-name" label="First name" />
        <MyyFormTextInput name="last-name" label="Last name" />
        <button>Submit form</button>
      </form>
    </FormContext>
  )
}
```

Clicking the button will trigger a submit action on the form. This is handled by the `handleSubmit` helper function. It will also ensure that the default submit event on the form tag itself is not triggered causing a page refresh.

Triggering submit programmatically can easily be done by calling `submit()` on the form object directly. Below is an example of triggering the form submit directly from a button. Note that it has the `type="button"` attribute in order to not trigger the default submit action on the enclosing form tag.

```tsx
import React from "react"
import { useForm, FormContext, handleSubmit } from "react-yafl"
import { submitForm, validateForm } from "./myFormLibrary" // You provide this part

export const MyForm = () => {
  const form = useForm({
    onSubmit: submitForm, // This function is memoized internally for performance reasons so it should be pure
    onValidate: validateForm, // This function is memoized internally for performance reasons so it should be pure
  })

  return (
    <FormContext form={form}>
      <form onSubmit={handleSubmit(form)}>
        <MyyFormTextInput name="first-name" label="First name" />
        <MyyFormTextInput name="last-name" label="Last name" />
        <button type="button" onClick={() => form.submit()}>
          Submit form
        </button>
      </form>
    </FormContext>
  )
}
```

### `onSubmit`

This function is called when the form control is submitted and no validation errors occur. It receives a single argument defined by the `IOnSubmitOptions` interface.

### `onValidate`

This function is responsible for returning an object containing the validation status of every field. It is triggered upon submit or whenever a field requests validation. It receives a single argument defined by the `IOnValidateOptions` interface. The function can return an object with error values for every field it validates. If an empty object or explicit `true` is returned the form is considered valid.

There are multiple ways that the `onValidate` function may be triggered:

1. When the form is submitted
2. When a field looses focus (depending on config)
3. When a field changes (depending on config)

For the latter two the `onValidate` function will also receive the name and current value of the field that triggered it. In these cases the validator is allowed to only validate the specific input field it recevied and may ignore other fields. See the `IOnValidateOptions` interface for details.

## Form Field API

To get the props you need use the hook `useFormField` in any component where you need access to the form.

```tsx
import React from "react"
import { getInputChangeValue } from "react-yafl/utils" // This util extracts the input value from a React.ChangeEvent (basically using event.target.value or event.target.checked (if type of target is "checkbox"))

interface IMyFormTextInputProps {
  name: string
  label: string
}

export const MyFormTextInput: React.FC<IMyFormTextInputProps> = ({
  name,
  label,
}) => {
  const { errors, value, onChange, onEnter, onLeave } = useFormField(name)

  return (
    <div>
      <label htmlFor={name}>{label}</label>
      <input
        type="text"
        id={name}
        name={name}
        value={value}
        onChange={getInputChangeValue(onChange)}
        onFocus={onEnter}
        onBlur={onLeave}
			/>
			{errors && (
				{errors.map(({id, message}) => (
					<p key={id}>{message}</p>
				))}
			)}
    </div>
  )
}
```

You may also use the `registerPrimitive` helper to automatically connect primitive components:

```tsx
import React from "react"
import { registerPrimitive } from "react-yafl/utils" // This util helps you connect all components that support common event props

interface IMyFormTextInputProps {
	name: string
	label: string
}

export const MyFormTextInput: React.FC<IMyFormTextInputProps> = ({ name, label }) => {
  const field = useFormField(name) // Uses context to retrieve field information
  // const field = useFormField<string>(name) // You can also define a type for the value

  return (
    <div>
      <label htmlFor={name}>{label}</label>
			<input type="text" {...registerPrimitive(name, field)} />
			{field.errors && (
				{field.errors.map(({id, message}) => (
					<p key={id}>{message}</p>
				))}
			)}
    </div>
  )
}
```

For component libraries you can create any kind of complex structure you want and simply hook up the relevat props as needed

```tsx
import React from "react"

import FormControlLabel from "@material-ui/core/FormControlLabel
import Checkbox from "@material-ui/core/Checkbox
import FormControl from "@material-ui/core/FormControl
import FormHelperText from "@material-ui/core/FormHelperText

interface IMyFormCheckbox {
	name: string
	label: string
}

export const MyFormCheckbox: React.FC<IMyFormCheckbox> = ({ name, label }) => {
	const { errors, value, onChange, onEnter, onLeave } = useFormField<boolean>(name) // Define a typed field.

	return (
		<FormControl error={!!errors}>
			<FormControlLabel
				label={label}
				control={
					<Checkbox
						id={name}
						name={name}
						checked={!!value} // Value is a boolean or undefined so we enforce the boolean value using a double negation.
						onChange={(evt: any, checked) => onChange(checked)} // Pass checked state to the change handler.
						inputProps={{
							onFocus: onEnter,
							onBlur: onLeave
						}}
					/>
				}
			/>
			{hasMessage && (
				<FormHelperText id={`${id}_helpertext`}>{message}</FormHelperText>
			)}
		</FormControl>
	)
}
```

### `value`

The `value` prop contains the real value of the form field. It can be any type you need it to be in order to support both simple and complex components. If the value has never been set or initialized it becomes `undefined`

### `onChange`

The `onChange` prop is a function that should be called whenever you wish to update the value of the form field. It takes the raw value as its only argument.

### `onEnter`

This function should be triggered whenever the user enters a form field. This usually corresponds to the `onFocus` event though for complex fields this may not be the case. Some "fields" may contian multiple, raw inputs thus the event can be triggered by the user accessing any of the corresponding elements on the page. Calling this multiple times is safe and will simply be ignored.

### `onLeave`

This function should be called whenever the user leaves the form field. Usually this is done through an `onBlur` event handler, but it may be triggered through any type of event that indicates the user has left the form element. Because some fields may contain complex values comprised of multiple, raw input fields this method is not called simply `onBlur` as leaving focus on one input may not technically indicate that the user has left the field entirely. Calling this function multiple times is safe and will simply be ignored.

### `doValidate`

This function triggers an immediate validation of this field and, depending on how you configure your validator, the entire form.

### `doReset`

This function resets the value of the field to its initial value. If no initial value was specified the value will be `undefined`.
