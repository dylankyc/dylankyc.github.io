# Render all chartjs charts in react typescript tailwindcss projects

In this tutorial, we'll show you how to render all chartjs charts in React project.

# Table of Contents

<!-- toc -->

# Define all chartjs charts

We copy all chartjs official example codes and put them in `component/chartjsofficial` directory. We also define title and component for each example chart.

```jsx
import { VerticalBarChart } from "@/component/chartjsofficial/verticalbarchart";
import { HorizontalBarChart } from "@/component/chartjsofficial/horizontalbarchart";
import { StackedBarChart } from "@/component/chartjsofficial/stackedbarchart";
import { GroupedBarChart } from "@/component/chartjsofficial/groupedbarchart";
import { AreaChart } from "@/component/chartjsofficial/areachart";
import { LineChart } from "@/component/chartjsofficial/linechart";
import { MultiAxisLineChart } from "@/component/chartjsofficial/multiaxislinechart";
import { PieChart } from "@/component/chartjsofficial/piechart";
import { DoughnutChart } from "@/component/chartjsofficial/doughnutchart";
import { PolarAreaChart } from "@/component/chartjsofficial/polarareachart";
import { RadarChart } from "@/component/chartjsofficial/radarchart";
import { ScatterChart } from "@/component/chartjsofficial/scatterchart";
import { BubbleChart } from "@/component/chartjsofficial/bubblechart";
import { MultiTypeChart } from "@/component/chartjsofficial/multitypechart";
import { ChartEvents } from "@/component/chartjsofficial/chartevents";
import { ChartRef } from "@/component/chartjsofficial/chartref";
import { GradientChart } from "@/component/chartjsofficial/gradientchart";
import { ChartEventsSingleDataset } from "@/component/chartjsofficial/charteventssingledataset";
import { ChartEventsSingleDatasetOutsideDatasource } from "@/component/chartjsofficial/charteventssingledatasetoutsidedatasource";

const components = [
  {
    title: "VerticalBarChart",
    component: VerticalBarChart,
  },
  {
    title: "HorizontalBarChart",
    component: HorizontalBarChart,
  },
  {
    title: "StackedBarChart",
    component: StackedBarChart,
  },
  {
    title: "GroupedBarChart",
    component: GroupedBarChart,
  },
  {
    title: "AreaChart",
    component: AreaChart,
  },
  {
    title: "LineChart",
    component: LineChart,
  },
  {
    title: "MultiAxisLineChart",
    component: MultiAxisLineChart,
  },
  {
    title: "DoughnutChart",
    component: DoughnutChart,
  },
  {
    title: "PolarAreaChart",
    component: PolarAreaChart,
  },
  {
    title: "RadarChart",
    component: RadarChart,
  },
  {
    title: "ScatterChart",
    component: ScatterChart,
  },
  {
    title: "BubbleChart",
    component: BubbleChart,
  },
  {
    title: "ScatterChart",
    component: ScatterChart,
  },
  {
    title: "MultiTypeChart",
    component: MultiTypeChart,
  },
  {
    title: "ChartEvents",
    component: ChartEvents,
  },
  {
    title: "ChartRef",
    component: ChartRef,
  },
  {
    title: "GradientChart",
    component: GradientChart,
  },
  {
    title: "ChartEventsSingleDataset",
    component: ChartEventsSingleDataset,
  },
];
```

# Write a funcitonal component

In order to render all components, we write a functional component to take array of component with title and render them in one place.

```jsx
type Component = {
  title: string,
  component: React.FunctionComponent,
};
type ChartProps = {
  components: Component[],
};

const ComponentWrapper: React.FC<ChartProps> = ({ components }) => {
  return (
    <div>
      {components.map((component, index) => {
        return (
          <ChartWrapper key={index} title={component.title}>
            <component.component />
          </ChartWrapper>
        );
      })}
    </div>
  );
};
```

# Write a chartjs wrapper

In order to add title for each chart, we write a functional component to wrapper up every chart with a `h1`. Note we define the `h1` style using tailwindcss, setting div size(`h-96`) and text size(`text-3xl`), etc.

```jsx
export const ChartWrapper: React.FC<{
  title: string,
  children: React.ReactNode,
}> = ({ title, children }) => {
  return (
    <div className="max-h-96 h-96  bg-slate-50 border border-dashed">
      <h1 className="text-3xl text-center font-bold underline">{title}</h1>
      {children}
    </div>
  );
};
```

# Final Code

First, import all chartjs components and define components to render.

```jsx
import { VerticalBarChart } from "@/component/chartjsofficial/verticalbarchart";
import { HorizontalBarChart } from "@/component/chartjsofficial/horizontalbarchart";
import { StackedBarChart } from "@/component/chartjsofficial/stackedbarchart";
import { GroupedBarChart } from "@/component/chartjsofficial/groupedbarchart";
import { AreaChart } from "@/component/chartjsofficial/areachart";
import { LineChart } from "@/component/chartjsofficial/linechart";
import { MultiAxisLineChart } from "@/component/chartjsofficial/multiaxislinechart";
import { PieChart } from "@/component/chartjsofficial/piechart";
import { DoughnutChart } from "@/component/chartjsofficial/doughnutchart";
import { PolarAreaChart } from "@/component/chartjsofficial/polarareachart";
import { RadarChart } from "@/component/chartjsofficial/radarchart";
import { ScatterChart } from "@/component/chartjsofficial/scatterchart";
import { BubbleChart } from "@/component/chartjsofficial/bubblechart";
import { MultiTypeChart } from "@/component/chartjsofficial/multitypechart";
import { ChartEvents } from "@/component/chartjsofficial/chartevents";
import { ChartRef } from "@/component/chartjsofficial/chartref";
import { GradientChart } from "@/component/chartjsofficial/gradientchart";
import { ChartEventsSingleDataset } from "@/component/chartjsofficial/charteventssingledataset";
import { ChartEventsSingleDatasetOutsideDatasource } from "@/component/chartjsofficial/charteventssingledatasetoutsidedatasource";

const components = [
  {
    title: "VerticalBarChart",
    component: VerticalBarChart,
  },
  {
    title: "HorizontalBarChart",
    component: HorizontalBarChart,
  },
  {
    title: "StackedBarChart",
    component: StackedBarChart,
  },
  {
    title: "GroupedBarChart",
    component: GroupedBarChart,
  },
  {
    title: "AreaChart",
    component: AreaChart,
  },
  {
    title: "LineChart",
    component: LineChart,
  },
  {
    title: "MultiAxisLineChart",
    component: MultiAxisLineChart,
  },
  {
    title: "DoughnutChart",
    component: DoughnutChart,
  },
  {
    title: "PolarAreaChart",
    component: PolarAreaChart,
  },
  {
    title: "RadarChart",
    component: RadarChart,
  },
  {
    title: "ScatterChart",
    component: ScatterChart,
  },
  {
    title: "BubbleChart",
    component: BubbleChart,
  },
  {
    title: "ScatterChart",
    component: ScatterChart,
  },
  {
    title: "MultiTypeChart",
    component: MultiTypeChart,
  },
  {
    title: "ChartEvents",
    component: ChartEvents,
  },
  {
    title: "ChartRef",
    component: ChartRef,
  },
  {
    title: "GradientChart",
    component: GradientChart,
  },
  {
    title: "ChartEventsSingleDataset",
    component: ChartEventsSingleDataset,
  },
];
```

Next, write a function called `ChartWrapper` to wrapper up react chartjs chart components with a `h1` as the title.

```jsx
export const ChartWrapper: React.FC<{
  title: string,
  children: React.ReactNode,
}> = ({ title, children }) => {
  return (
    <div className="max-h-96 h-96  bg-slate-50 border border-dashed">
      <h1 className="text-3xl text-center font-bold underline">{title}</h1>
      {children}
    </div>
  );
};
```

Then, write a functional component to take a arrays of components and render them use `ChartWrapper` component.

```jsx
type Component = {
  title: string,
  component: React.FunctionComponent,
};
type ChartProps = {
  components: Component[],
};

const ComponentWrapper: React.FC<ChartProps> = ({ components }) => {
  return (
    <div>
      {components.map((component, index) => {
        return (
          <ChartWrapper key={index} title={component.title}>
            <component.component />
          </ChartWrapper>
        );
      })}
    </div>
  );
};
```

Finally, in `App` component, we iterate the components and render them in a div with `grid` class.

```jsx
export default function App() {
  return (
    <div>
      <Head>
        <title>ChartJS + NextJS + TailwindCSS</title>
        <meta name="description" content="Generated by create next app" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <link rel="icon" href="/favicon.ico" />
      </Head>
      <main className={styles.main}>
        <div className={styles.description}>
          <h1 className="text-3xl text-center font-bold underline">
            ChartJS + NextJS + TailwindCSS
          </h1>
          <div className="container columns-2">
            <div className="grid grid-cols-2 md:grid-cols-1 gap-0">
              {<ComponentWrapper components={components} />}
            </div>
          </div>
        </div>
      </main>
    </div>
  );
}
```

# Demo

![chartjs-react-chartjs-2-nextjs-tailwind-typescript-using-grid](https://user-images.githubusercontent.com/74223747/224673738-9edb7c6d-5e49-44a2-9bb0-00fa9eb5f608.png)

# Optimize

Note, for `ComponentWrapper` function component, it's more verbose. We could write it in another way.

Here is the old way.

```jsx
// NOTE: OK but verbose
const ComponentWrapper: React.FC<ChartProps> = ({ components }) => {
  return (
    <div>
      {components.map((c, index) => {
        return (
          <ChartWrapper key={index} title={c.title}>
            <c.component />
          </ChartWrapper>
        );
      })}
    </div>
  );
};
```

The new way:

```jsx
// NOTE: concise
const ComponentWrapperChatGPTAgain: React.FC<ChartProps> = ({ components }) => {
  return (
    <div>
      {components.map((c, index) => {
        const { title, component: C } = c;

        return (
          <ChartWrapper key={index} title={title}>
            <C />
          </ChartWrapper>
        );
      })}
    </div>
  );
};
```

To render list of charts components with titles and properties, we wrote two React component functions. While both functions produce the same output, their implementation approaches are different.

The first function `ComponentWrapper` is a good method, but the downside is that it is quite verbose. It uses curly braces and a return statement to wrap the component inside `<ChartWrapper>`, and it also references `c.title` and `c.component` multiple times within the function body.

The second function `ComponentWrapperChatGPTAgain` is a more concise implementation. It uses ES6 destructuring syntax to get title and component from the components object, which avoids repeated use of `c.title` and `c.component`. Additionally, it uses a more descriptive variable name `C` for the component.

Clearly, the second implementation is more succinct, easier to read and maintain, and also makes it easier to add new properties or methods. Therefore, we recommend using the second function `ComponentWrapperChatGPTAgain` as a more efficient programming practice.

To summarize, the primary difference between these two implementation methods is that the first one is verbose, which makes it difficult to read and maintain, while the second one is more concise and easier to read and maintain. The second function also uses destructuring, which is a useful syntax for improving readability by eliminating repetition. By using a more concise and readable implementation, you can streamline your code and make it more efficient.

## chatgpt says

Comparing Two React Component Functions

In this post, we will compare and contrast two React component functions that are used to render a list of chart components with titles and properties. While both functions ultimately produce the same result, their implementation approaches are different.

Function 1: ComponentWrapper

```jsx
const ComponentWrapper: React.FC<ChartProps> = ({ components }) => {
  return (
    <div>
      {components.map((c, index) => {
        return (
          <ChartWrapper key={index} title={c.title}>
            <c.component />
          </ChartWrapper>
        );
      })}
    </div>
  );
};
```

This function is a good approach, but it is quite verbose. It uses curly braces and a return statement to wrap the component inside `<ChartWrapper>`, and it also references `c.title` and `c.component` multiple times within the function body. It might be a bit difficult to read and maintain for developers, especially with more complex components.

Function 2: ComponentWrapperChatGPTAgain

```jsx
const ComponentWrapperChatGPTAgain: React.FC<ChartProps> = ({ components }) => {
  return (
    <div>
      {components.map((c, index) => {
        const { title, component: C } = c;

        return (
          <ChartWrapper key={index} title={title}>
            <C />
          </ChartWrapper>
        );
      })}
    </div>
  );
};
```

This function, on the other hand, is a more concise implementation. It uses ES6 destructuring syntax to get title and component from the components object, which avoids repeated use of `c.title` and `c.component`. Additionally, it uses a more descriptive variable name `C` for the component.

The second implementation is more succinct, easier to read and maintain, and also makes it easier to add new properties or methods in the future. Therefore, we recommend using the second function `ComponentWrapperChatGPTAgain` as a more efficient programming practice.

Conclusion

The primary difference between these two implementation methods is that the first one is verbose, which makes it difficult to read and maintain, while the second one is more concise and easier to read and maintain. The second function also uses destructuring, which is a useful syntax for improving readability by eliminating repetition. By using a concise, readable implementation, code can be streamlined and more efficient.
