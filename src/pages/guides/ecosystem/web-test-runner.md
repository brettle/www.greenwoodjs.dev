---
layout: guides
order: 5
tocHeading: 2
---

# Web Test Runner

[**Web Test Runner**](https://modern-web.dev/docs/test-runner/overview/) is a developer tool created by the [Modern Web](https://modern-web.dev/) team that helps with facilitating the testing Web Components, especially being able to test them in a real browser. This guide will give a high level over of setting up WTR and integrating with any Greenwood specific capabilities.

> You can see an example project (this website's own repo!) [here](https://github.com/ProjectEvergreen/www.greenwoodjs.dev).

## Setup

For the sake of this guide, we will be covering a minimal setup but you are free to extends things as much as you need.

1. First, let's install WTR and the JUnit Reporter. You can use your favorite package manager

   ```shell
   npm i -D @web/test-runner @web/test-runner-junit-reporter
   ```

1. You'll also want something like [**chai**](https://www.chaijs.com/) to write your assertions with

   ```shell
   npm i -D @esm-bundle/chai
   ```

1. Next, create a basic _web-test-runner.config.js_ configuration file

   ```js
   import { defaultReporter } from "@web/test-runner";
   import { junitReporter } from "@web/test-runner-junit-reporter";

   export default {
     // customize your spec pattern here
     files: "./src/**/*.spec.js",
     // enable this if you're using npm / node_modules
     nodeResolve: true,
     // optionally configure reporters and coverage
     reporters: [
       defaultReporter({ reportTestResults: true, reportTestProgress: true }),
       junitReporter({
         outputPath: "./reports/test-results.xml",
       }),
     ],
     coverage: true,
     coverageConfig: {
       reportDir: "./reports",
     },
   };
   ```

## Usage

With everything install and configured, you should now be good to start writing your tests! 🏆

```js
// src/components/footer/footer.js
export default class Footer extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <footer>
        <h4 class="heading">Greenwood</h4>
        <img src="/assets/my-logo.webp" />
      </footer>
    `;
  }
}

customElements.define("app-footer", Footer);
```

```js
// src/components/footer/footer.spec.js
describe("Components/Footer", () => {
  let footer;

  before(async () => {
    footer = document.createElement("app-footer");
    document.body.appendChild(footer);

    await footer.updateComplete;
  });

  describe("Default Behavior", () => {
    it("should not be null", () => {
      expect(footer).not.equal(undefined);
      expect(footer.querySelectorAll("footer").length).equal(1);
    });

    it("should have the expected heading", () => {
      const header = footer.querySelectorAll("footer .heading");

      expect(header.length).equal(1);
      expect(header[0].textContent).to.equal("Greenwood");
    });

    it("should have the expected logo image", () => {
      const logo = footer.querySelectorAll("footer img[src]");

      expect(logo.length).equal(1);
      expect(logo[0]).not.equal(undefined);
    });
  });

  after(() => {
    footer.remove();
    footer = null;
  });
});
```

## Static Assets

If you are seeing logging about static assets returning 404

```shell
 🚧 404 network requests:
    - assets/my-image.png
```

You can create a custom middleware in your _web-test-runner.config.js_ to resolve these requests to your local workspace:

```js
import path from "path";
// ...

export default {
  // ...

  middleware: [
    function resolveAssets(context, next) {
      const { url } = context.request;

      if (url.startsWith("/assets")) {
        context.request.url = path.join(process.cwd(), "src", url);
      }

      return next();
    },
  ],
};
```

## Resource Plugins

If you're using one of Greenwood's [resource plugins](/docs/plugins/), you'll need to customize WTR manually through [its plugins option](https://modern-web.dev/docs/test-runner/plugins/) so it can leverage the Greenwood plugins your using to automatically to handle these custom transformations.

For example, if you're using Greenwood's [Raw Plugin](https://github.com/ProjectEvergreen/greenwood/tree/master/packages/plugin-import-raw), you'll need to add a plugin transformation and stub out the signature.

```js
import fs from "fs/promises";
// 1) import the greenwood plugin and lifecycle helpers
import { greenwoodPluginImportRaw } from "@greenwood/plugin-import-raw";
import { readAndMergeConfig } from "@greenwood/cli/src/lifecycles/config.js";
import { initContext } from "@greenwood/cli/src/lifecycles/context.js";

// 2) initialize Greenwood lifecycles
const config = await readAndMergeConfig();
const context = await initContext({ config });
const compilation = { context, config };

// 3) initialize the plugin
const rawResourcePlugin = greenwoodPluginImportRaw()[0].provider(compilation);

export default {
  // ...

  // 4) add it the plugins option
  plugins: [
    {
      name: "import-raw",
      async transform(context) {
        const { url } = context.request;

        if (url.endsWith("?type=raw")) {
          const contents = await fs.readFile(new URL(`.${url}`, import.meta.url), "utf-8");
          const response = await rawResourcePlugin.intercept(null, null, new Response(contents));
          const body = await response.text();

          return {
            body,
            headers: { "Content-Type": "application/javascript" },
          };
        }
      },
    },
  ],
};
```

## Content as Data

If you are using any of Greenwood's [content as data](/docs/content-as-data/) features, you'll want to have your tests handle mocking of `fetch` calls. This can be done with a simple override of `window.fetch`

```js
import { expect } from "@esm-bundle/chai";
import graph from "../../stories/mocks/graph.json" with { type: "json" };
import "./blog-posts-list.js";

// override fetch to return a promise that resolves to our mock data
window.fetch = function () {
  return new Promise((resolve) => {
    resolve(new Response(JSON.stringify(graph)));
  });
};

// now we can test components as normal
describe("Components/Blog Posts List", () => {
  let list;

  before(async () => {
    list = document.createElement("app-blog-posts-list");
    document.body.appendChild(list);

    await list.updateComplete;
  });

  describe("Default Behavior", () => {
    it("should not be null", () => {
      expect(list).not.equal(undefined);
    });

    it("should render list items for all our blog posts", () => {
      expect(list.querySelectorAll("ul").length).to.be.equal(1);
      expect(list.querySelectorAll("ul li").length).to.be.greaterThan(1);
    });

    // ...
  });

  after(() => {
    list.remove();
    list = null;
  });
});
```

> To quickly get a "mock" graph to use in your stories, you can run `greenwood build` and copy the _graph.json_ file from the build output directory.