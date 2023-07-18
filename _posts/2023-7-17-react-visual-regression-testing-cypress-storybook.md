---
layout: single
title: "Visual Regression Testing with Cypress and Storybook"
excerpt: "Improve your React application's user experience by ensuring visual consistency across every update, using Cypress and Storybook with advanced techniques such as visual regression testing."
header:
  image: /assets/images/posts/react-visual-regression-testing-cypress-storybook/header.jpg
  teaser: /assets/images/posts/react-visual-regression-testing-cypress-storybook/header.jpg
  overlay_image: /assets/images/posts/react-visual-regression-testing-cypress-storybook/header.jpg
  overlay_filter: 0.5
  og_image: /assets/images/posts/react-visual-regression-testing-cypress-storybook/og.png
date: 2023-07-17
comments: true
toc: true
---

When we were introduced to unit testing with Jest, we've been told about
quitting comparing the existence, attributes and values of our DOM elements
with the ones we expect in our components.

These assertions have been replaced by snapshots, which are a representation of
the DOM at a given time.

```tsx
import { expect, test } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import App from "./App.tsx";

test("displays initial counter", async () => {
  render(<App />);

  // expect(screen.getByRole("button").textContent).toEqual("count is 0");
  expect(screen.getByRole("button")).toMatchSnapshot();
});
```

Running the test above will create a snapshot file in the `src/__snapshots__`
folder, named `App.test.tsx.snap`.

This file contains the DOM representation of the component at the time the test
was run, with the expected values.

```tsx
// Vitest Snapshot v1, https://vitest.dev/guide/snapshot.html

exports[`displays initial counter 1`] = `
<button>
  count is 
  0
</button>
`;
```

The test would succeed if the snapshot file doesn't exist, or if the DOM
representation matches the one in the existing snapshot file.

Modifying the component would fail the test, and the snapshot file would have to
be updated with the new DOM representation.

```
 FAIL  src/App.test.tsx > displays initial counter
Error: Snapshot `displays initial counter 1` mismatched
 ❯ src/App.test.tsx:11:38
      9| 
     10|   // expect(screen.getByRole("button").textContent).toEqual("count is 0");
     11|   expect(screen.getByRole("button")).toMatchSnapshot();
       |                                      ^
     12| });
     13| 

  - Expected  - 1
  + Received  + 1

    `<button>␊
  -   count is ␊
  +   Count value is ␊
      0␊
    </button>`
```

You could just run `npm test -- -u` to confirm that your modification was
intentional, update the snapshot file and make the test pass again.

It is now guaranteed that our component is **rendered** as expected.

Despite the apparent convenience and reliability of snapshots, one issue
remains: they guarantee that our component is rendered as expected, but not that
it will look as expected. For that, we need a different testing approach.

# Enter Visual Regression Testing

Visual Regression Testing is a technique that allows you to compare the rendered
component with a reference image, and fail the test if the two images don't
match.

Instead of taking a snapshot of the DOM representation of the component, we
want to take a snapshot of the rendered component and store it as an image,
like if we were taking a screenshot.

<figure>
  <a href="/assets/images/posts/react-visual-regression-testing-cypress-storybook/original.jpg">
    <img src="/assets/images/posts/react-visual-regression-testing-cypress-storybook/original.jpg">
  </a>
  <figcaption>Original component</figcaption>
</figure>

Then our `toMatchSnapshot` assertion would compare the rendered component with
the reference image, fail the test if the two images don't match and show us
the difference between the two images.

<figure>
  <a href="/assets/images/posts/react-visual-regression-testing-cypress-storybook/diff.jpg">
    <img src="/assets/images/posts/react-visual-regression-testing-cypress-storybook/diff.jpg">
  </a>
  <figcaption>Comparison between the original and modified versions</figcaption>
</figure>

How can we do this magic, knowing that our tests are running inside a terminal
and simulated DOM environment thanks to `jsdom`?

We will have to use a web browser to render our component, which is good news
since most of our co-workers have at least Chrome or Firefox installed on their
computer.

However, the bad news is that our co-workers and other developers around the
world have different screen resolutions, operating systems and typography
settings which can greatly impact on the way our component is rendered and give
us false positives.

In order to tackle this issue, we will have to use a single web browser,
with a fixed screen resolution and same fonts installed to run our tests. The
best solution we have at the time of writing this article is to use a Docker
image with a headless browser installed.

You have to keep in mind that the resulting docker image might be quite heavy
to download and store inside your private docker registry, but it's still worth
it if you want to avoid the design of your application to be broken by a
single modification in a global CSS file.

Also having a whole web browser stored inside your docker image makes sense
since that same web browser is your target platform, as your users will be
using it to access your application.

## Docker image

Once you've convinced your infra team to store a (maximum of) 2 GB docker image,
for dependencies caching purposes, in their private docker registry, you can
start working on the docker image itself.

Fortunately for us, there are already [some docker images][cypress-docker]
with Cypress, Node and the Google Chrome web browser pre-installed.

Create a new `Dockerfile` at the root of your application with the following
contents:

```dockerfile
FROM cypress/browsers:node18.12.0-chrome107

WORKDIR /app

COPY package.json package-lock.json /app/
RUN npm ci

COPY . /app
RUN npm run build
```

Now you can build this image by running the following command:

```bash
$ docker build -t my-app .
```

And execute Cypress inside the container:

```bash
$ docker run -it my-app npm run test:e2e
```

Feel free to mount some directories to store the screenshots and videos in
case you would like to try failing some tests:

```bash
$ docker run -it \
  -v `pwd`/cypress/screenshots:/app/cypress/screenshots \
  -v `pwd`/cypress/videos:/app/cypress/videos \
  my-app sh -c "npm run test:e2e ; chown -R $(id -u):$(id -g) cypress"
```

Our docker image is now ready to be used in our CI pipeline. Let's see how we
can perform visual regression testing with Cypress.

## Cypress

You will find commercial options available to setup visual regression testing
in a seamless way [on the official website][plugins], but let's focus on the
open-source ones.

I've been using `cypress-image-snapshot` during the last few years, but it seems
unmaintained at the time of writing this article. After digging a bit, I found
a fork of this plugin called
[cypress-plugin-visual-regression-diff][cypress-plugin-visual-regression-diff],
made by the folks at [FRSource][frsource].

The installation instructions are very straightforward with very low effort
required. First, you have to get the package:

```bash
$ npm install --save-dev @frsource/cypress-plugin-visual-regression-diff
```

Then initialize it inside your `cypress.config.ts` file:

```diff
 import { defineConfig } from "cypress";
+import { initPlugin } from "@frsource/cypress-plugin-visual-regression-diff/plugins";
 
 export default defineConfig({
   e2e: {
     baseUrl: "http://localhost:3000",
     setupNodeEvents(on, config) {
       // implement node event listeners here
+      initPlugin(on, config);
     },
   },
 
   component: {
     devServer: {
       framework: "react",
       bundler: "vite",
     },
+    setupNodeEvents(on, config) {
+      initPlugin(on, config);
+    },
   },
 });
```

Register the commands inside your `cypress/support/commands.ts` file:

```ts
import "@frsource/cypress-plugin-visual-regression-diff";
```

That's it. Inside your test file, located at `cypress/e2e/App.cy.ts` you can use
the `cy.matchImage();` method to take a snapshot of the rendered component and
compare it with the reference image automatically.

```diff
 describe("default vite react app", () => {
   it("increments the counter", () => {
     cy.visit("/");
     cy.get("button").should("have.text", "count is 0");
     cy.get("button").click().should("have.text", "count is 1");
+    cy.matchImage();
   });
 });
```

The next time you'll run your tests with `npm run test:e2e`, the following
output will be displayed:

```
  (Screenshots)

  -  /home/unnamedcoder/git/my-app/cypress/e2e/__image_snapshots__/default vite react app inc     (1000x660)
     rements the counter #0.actual.png 
```

As you can see, we've just created an image snapshot of our whole page. The
file located inside the `cypress/e2e/__image_snapshots__` directory should be
committed to your git repository.

But before we are doing such thing, we need to understand why we've created
a docker image previously, by running the same tests with docker this time:

```bash
$ docker build -t my-app .
$ docker run -it \
  -v `pwd`/cypress/e2e/__image_snapshots__:/app/cypress/e2e/__image_snapshots__ \
  -v `pwd`/cypress/screenshots:/app/cypress/screenshots \
  -v `pwd`/cypress/videos:/app/cypress/videos \
  my-app sh -c "npm run test:e2e ; chown -R $(id -u):$(id -g) cypress"
```

A similar error to the following one below should be outputted:

```
  1) default vite react app
       increments the counter:
     Error: Image diff factor (0.989%) is bigger than maximum threshold option 0.01.
      at Context.eval (webpack:///./node_modules/@frsource/cypress-plugin-visual-regression-diff/dist/support.js:154:0)
```

This means that the rendered component is different from the reference image by
more than 1%. This is due to the fact that the docker image is using a different
operating system, screen resolution and fonts than your local machine.

Since that docker image will be used by our CI pipeline, we need to make sure
that the reference image is the same as the one generated by the docker image.
And tell our co-workers to run the tests exclusively with docker to avoid
having different reference images.

Therefore, we need to update the reference images generated by ourselves, with
the ones generated by the docker image.

Since there's actually no need to remove it by hand, we need to add the
following lines into the `scripts` section of our `package.json` file:

```diff
   "scripts": {
     "dev": "vite",
     "build": "tsc && vite build",
     // ...
     "test:e2e-start": "cypress run --e2e",
     "test:e2e": "start-server-and-test dev http-get://localhost:3000 test:e2e-start",
+    "test:e2e-update-start": "cypress run --e2e --env pluginVisualRegressionUpdateImages=true",
+    "test:e2e-update": "start-server-and-test dev http-get://localhost:3000 test:e2e-update-start",
     // ...
   },
```

And run our new `test:e2e-update` script we've just added:

```bash
$ docker build -t my-app .
$ docker run -it \
  -v `pwd`/cypress/e2e/__image_snapshots__:/app/cypress/e2e/__image_snapshots__ \
  -v `pwd`/cypress/screenshots:/app/cypress/screenshots \
  -v `pwd`/cypress/videos:/app/cypress/videos \
  my-app sh -c "npm run test:e2e-update ; chown -R $(id -u):$(id -g) cypress"
```

Our tests should have passed and the reference image should have been updated.

## What about Storybook ?

Visual regression testing with Storybook is pretty straightforward. I've been
using Loki for a while now but the latest version seems to have trouble with
Storybook 7 and React 18.

Which is why we're going to switch to Storycap for generating the Storybook
snapshots, with the help of reg-cli for comparing them.

Storycap can be installed with the following command:

```bash
$ npm install storycap --save-dev
```

Snapshot generation will be done with the command below:

```bash
$ storycap --serverCmd "storybook dev -p 9001" http://localhost:9001

info Wait for connecting storybook server http://localhost:9001.
info Executable Chromium path: /usr/bin/google-chrome-stable
info Storycap runs with simple mode
info Found 1 stories.
info Screenshot stored: __screenshots__/App/Default.png in 573 msec.
info Screenshot was ended successfully in 10826 msec capturing 1 PNGs.
```

As stated in the output above, the generated snapshot is located inside the
`__screenshots__` directory.

Let's install reg-cli to perform the visual regression testing:

```bash
$ npm install reg-cli --save-dev
```

reg-cli will take as first parameter the path to the folder containing the
actual images we will generate during the test (in our case, `__screenshots__`),
the folder containing our reference images as second parameter and the folder
where the diff images will be stored as third parameter.

It can also generate a report in HTML format with the `-R` parameter, which is
very convenient for debugging purposes when our tests are failing.

First, we can ask reg-cli to store our generated snapshots as reference images:

```bash
$ reg-cli ./__screenshots__ ./expected -U
✔ pass    __screenshots__/App/Default.png

All images are updated. 
✨ your expected images are updated ✨


✔ 1 file(s) passed.
```

Then, we can run our tests with the following command:

```bash
$ reg-cli ./__screenshots__ ./expected ./diff
✔ pass    __screenshots__/App/Default.png


✔ 1 file(s) passed.
```

As a reminder, these commands should be run through Docker as well, to make
sure that the reference images are the same as the ones generated by our CI
pipeline.

You can add the following scripts inside your `package.json` file:

Since there's actually no need to remove it by hand, we need to add the
following lines into the `scripts` section of our `package.json` file:

```diff
   "scripts": {
     "dev": "vite",
     "build": "tsc && vite build",
     // ...
     "test:e2e-start": "cypress run --e2e",
     "test:e2e": "start-server-and-test dev http-get://localhost:3000 test:e2e-start",
+    "test:visual": "reg-cli ./__screenshots__ ./expected ./diff",
+    "test:visual-capture": "storycap --serverCmd \"storybook dev -p 9001\" http://localhost:9001",
+    "test:visual-update": "reg-cli ./__screenshots__ ./expected -U",
     // ...
   },
```

Run the following command to (re)generate the reference images:

```bash
$ docker build -t my-app .
$ docker run -it \
  -v `pwd`/__screenshots__:/app/__screenshots__ \
  -v `pwd`/expected:/app/expected \
  -v `pwd`/diff:/app/diff \
  my-app sh -c "npm run test:visual-capture && npm run test:visual-update ; chown -R $(id -u):$(id -g) __screenshots__ expected"
```

And run our new `test:visual` script we've just added to make sure our design
remains intact:

```bash
$ docker run -it \
  -v `pwd`/__screenshots__:/app/__screenshots__ \
  -v `pwd`/expected:/app/expected \
  -v `pwd`/diff:/app/diff \
  my-app sh -c "npm run test:visual ; chown -R $(id -u):$(id -g) __screenshots__ expected diff"
```

# Wrapping up

In conclusion, visual regression testing represents a significant step forward
in ensuring the integrity of our application's design. By embracing this method,
we can avoid unwanted changes in design caused by minor alterations in global
CSS files or dependencies updates.

Despite certain challenges, such as maintaining large Docker images and
adjusting to local machine differences, the overall benefits, particularly in
maintaining UI consistency, are well worth the investment.

Therefore, developers and teams should consider visual regression testing as an
indispensable component in their testing toolkit.

[cypress-docker]: https://github.com/cypress-io/cypress-docker-images
[plugins]: https://docs.cypress.io/plugins#Visual%20Testing
[cypress-plugin-visual-regression-diff]: https://github.com/FRSOURCE/cypress-plugin-visual-regression-diff
[frsource]: https://www.frsource.org/