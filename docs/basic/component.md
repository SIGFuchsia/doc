# Components

According to [Fuchsia's Guides: Develop components](https://fuchsia.dev/fuchsia-src/development/components/build)

> At runtime, **Component instances** see the contents of their package as read-only files under the path `/pkg`. Defining two or more components in the same package doesn't grant each component access to the other's capabilities. However it can guarantee to one component that the other is available. Therefore if a component attempts to launch an instance of another component, such as in an integration test, it can be beneficial to package both components together.
>
> Components are instantiated in a few ways, all of which somehow specify their **URL**. Typically components are launched by specifying their package names and path to their component manifest in the package, using the `fuchsia-pkg://scheme`.


