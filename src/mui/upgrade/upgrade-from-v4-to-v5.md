# Title: Upgrading Material-UI from v4 to v5: A Comprehensive Guide

<!-- toc -->

# Introduction:

Material-UI is a popular React component library that provides a set of pre-built UI components for building modern and responsive web applications. With the release of Material-UI v5, there have been significant changes and improvements, making it essential for developers to upgrade from v4 to v5. In this blog post, we will walk you through the step-by-step process of upgrading Material-UI to its latest version.

# Step 1: Upgrade React to 17.0.0:

To start the upgrade process, it is necessary to update React to version 17.0.0 or above. This can be done using the following command:

```bash
yarn upgrade @material-ui/core@^4.11.2 react@^17.0.0
```

# Step 2: Update MUI packages and peer dependencies:

Next, we need to update the Material-UI packages and their peer dependencies. Run the following commands to install the required packages:

```bash
yarn add @mui/material @mui/styles
yarn add @mui/lab
yarn add @mui/icons-material
yarn add @emotion/react @emotion/styled
```

# Step 3: Run codemods:

Material-UI provides codemods that automatically adjust your code to account for breaking changes in v5. These codemods help in migrating your codebase efficiently. Run the following command to apply the preset-safe codemod:

```bash
npx @mui/codemod v5.0.0/preset-safe <path>
```

Additionally, you can run specific codemods for individual components or pages if needed. For example:

```bash
npx @mui/codemod v5.0.0/preset-safe components
npx @mui/codemod v5.0.0/preset-safe pages
```

# Step 4: Fix broken code:

After running the codemods, it's important to review your codebase for any broken code. One common issue is the usage of the `theme.spacing()` function, which has changed in v5. Replace instances of `theme.spacing(2)` with `2`, `theme.spacing(4)` with `4`, and so on, to fix this issue.

# Step 5: Replace all imports:

With the release of v5, the package names have changed from `@material-ui/*` to `@mui/*`. To ensure compatibility with the latest version, replace all imports accordingly. Here are some examples:

```bash
yarn remove @material-ui/core
yarn remove @material-ui/icons
yarn remove @material-ui/lab
yarn remove @material-ui/pickers

yarn remove @mui/x-data-grid
yarn add @mui/x-data-grid
```

# Step 6: Test and finalize:

After completing the above steps, thoroughly test your application to ensure that it runs without any errors. Make any necessary adjustments or fixes as required. Once you are confident that your application is functioning correctly, commit the changes and finalize the upgrade process.

Conclusion:
Upgrading Material-UI from v4 to v5 is an important step to take advantage of the latest features, bug fixes, and improvements. By following the steps outlined in this guide, you can smoothly upgrade your Material-UI-based application to the latest version. Remember to thoroughly test your application after the upgrade to ensure everything is functioning as expected. Happy coding with Material-UI v5!

# References:

- Material-UI Migration Guide: [Migrating to v5: getting started](https://mui.com/material-ui/migration/migration-v4/)
