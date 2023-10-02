# Declarative Integrated Search

This is a guide for experimenting with search in a declarative integrated Backstage frontend application.

> **Disclaimer:** Currently, it is in an experimental stage and is not recommended for production

Declarative integration is an way of customizing your Backstage instance without code, but only with configuration, see this RFC for more details.

The main principle of declarative integration is to install Backstage plugins with default extensions that can be customized by adopters via configuration file.
In this new architecture, everything that extends the core Backstage features is called a extension, so extension could be since an api to a page. As en example, a backstage instance
can be extended with a search plugin that is a collection of extensions for implementing and customization Backstage search capability.

As you can read on this [RFC](https://github.com/backstage/backstage/issues/18372), extensions:

- Can have outputs: this is the artifact that is produced by an extension, it could be an metadata data, api implementation, a component element and etc.
- Could expect inputs: These are extensions outputs that are injected as input of others;
- Rendered in attachment point: The attachment represents another extension input, that means that an extension output will be injected as an input of the extension referred as the attachment point.

The core concepts mentioned briefly above are crucial for understanding the the declarative version of the `Search` plugin.

## Search Plugin

### Introduction

The search plugin currently provides the following extensions:

- SearchApi: this is a web client abstraction for communicating with the Backstage Search backend, the output of this extension is an concrete implementation for the Search API interface and the output is attached as an input to the core plugin apis holder;
- SearchResultItem: It is an extension that outputs a component responsible for rendering search result list item, when installed, they are passed to the search page as inputs.
- SearchPage: It is a component that represents the interface for the advanced Search page, this extension expects search result items components as optional inputs so it uses tem to render search results in a custom way.
- SearchNavItem: it is an extension that output the data to represent a search item in the main application sidebar, in other other, it is passed to the core nav extension as an input.

### Installation

Now only one step is required to start using the Search plugin in Backstage, which is to install the `@backstage/plugin-catalog` and `@backstage/plugin-search` packages to a Backstage application that supports declarative integration (e.g., [`app-next`](https://github.com/backstage/backstage/tree/master/packages/app-next)).

```sh
yarn add @backstage/plugin-catalog @backstage/plugin-search
```

### Configuration

The experimental Search page is configurable via `app-config.yaml` file:

```yaml
# app-config.yaml
# example disabling analytics tracking
app:
  experimental:
    packages: 'all' # ✨

  extensions:
    - plugin.search.page
         config:
           noTrack: true

```

### Contribution

Plugin developers can use the `createSearchResultItemExtension` factory provided by the `@backstage/plugin-search-react` for building custom result item extensions:

```tsx
// plugins/techdocs/index.ts

/** @alpha */
export const TechDocsSearchResultListItemExtension =
  createSearchResultListItemExtension({
    id: 'techdocs',
    configSchema: createSchemaFromZod(z =>
      z.object({
        noTrack: z.boolean().default(false),
        lineClamp: z.number().default(5),
      }),
    ),
    predicate: result => result.type === 'techdocs',
    component: async ({ config }) => {
      const { TechDocsSearchResultListItem } = await import(
        './components/TechDocsSearchResultListItem'
      );
      return props => <TechDocsSearchResultListItem {...props} {...config} />;
    },
  });

/** @alpha */
export default createPlugin({
  id: 'techdocs',
  extensions: [TechDocsSearchResultListItemExtension],
});
```

In the snippet above, a TechDocs plugin developer is providing a custom search result component for Search results of type "techdocs" by default.

In case an adopter doesn't want to use the custom TechDocs search result item, they can disabled it via Backstage configuration file.

```yaml
# app-config.yaml
app:
  experimental:
    packages: 'all' # ✨

  extensions:
    - plugin.search.result.item.techdocs: false
```

Also because a configuration schema as provided while the TechDocs search result extension was created, that means backstage adopters can costumized TechDocs search results line clamp that defaults to 3 and can also disable automatic analytic events tracking.
For doing so, in the `app-config.yaml`.

```
# app-config.yaml
app:
  experimental:
    packages: 'all' # ✨

  extensions:
    - plugin.search.result.item.techdocs:
        config:
          noTrack: true
          lineClamp: 3
```

The `createSearchResultItemExtension` outputs a representation of an extension in Backstage as follows:

```ts
{
  "$$type": "@backstage/Extension",
  "id": "plugin.search.result.item.techdocs",
  "at": "plugin.search.page/items",
  "configSchema": {
    "schema": {
      "type": "object",
      "properties": {
        "noTrack": {
          "type": "boolean",
          "default": false
        },
        "lineClamp": {
          "type": "number",
          "default": 5
        }
      },
      "additionalProperties": false,
      "$schema": "http://json-schema.org/draft-07/schema#"
    }
  },
  "output": {
      "item": {
          "id": "plugin.search.result.item.data",
          "$$type": "@backstage/ExtensionDataRef",
          "config": {}
      }
  },
  "disabled": false,
  "inputs": {}
}
```

- id: The extension id is `plugin.search.result.item.techdocs` and it is the same key used to configure it in the `app-config.yaml` file/
- at: It represents the extension attachment point, so the value `plugin.search.page/items` says that the TechDocs search result output will be injected in the items inputs expected by the search page.
- configurationSchema: represents the TechDocs search result item configuration schema;
- output: Is the type of the artifact produced by the TechDocs result item, whit is a react component;
- inputs: in this case is an empty object because this extension doesn't not expect outputs of any other;
- disable: Says that the result item extension will be enable by default when the TechDocs plugin is installed in the app.

TODO: write about installing the custom result item.
