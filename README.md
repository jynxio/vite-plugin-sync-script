## Description

During the build phase, use Vite's existing configuration to process all synchronous JavaScript code in `index.html`, for example:

```html
<!-- 🥩 Before -->
<head>
    <script src="./sync-script-1.js"></script>
    <script>/* sync-script-2 */</script>
</head>
<body>
    <script>/* sync-script-3 */</script>
</body>

<!-- 🍖 After -->
<head>
    <script>
        // sync-script-1, sync-script-2, sync-script-3... After compression, transpilation, etc.
    </script>
</head>
```

> Only JavaScript syntax is supported.

## Implementation

```ts
import path from "node:path";
import crypto from "node:crypto";
import fs from "node:fs";
import { JSDOM } from "jsdom";

export default function extraEntryPlugin() {
    let allScriptElemCode = "";

    const allScriptElemContent: string[] = [];
    const virtualFileId = `virtual:${crypto.randomUUID()}`;
    const resolvedVirtualFileId = `\0${virtualFileId}`;
    const { promise, resolve } = Promise.withResolvers();

    return [
        {
            name: "vite-plugin-sync-script-pre",
            apply: "build",
            resolveId(id) {
                if (id === virtualFileId) return resolvedVirtualFileId;
            },
            load(id) {
                if (id === resolvedVirtualFileId) return promise;
            },
            config(config) {
                config.build = {
                    ...config.build,
                    rollupOptions: {
                        ...config.build?.rollupOptions,
                        input: {
                            ...(config.build?.rollupOptions?.input || {
                                index: path.resolve(__dirname, "index.html"),
                            }),
                            [virtualFileId]: virtualFileId,
                        },
                    },
                };
            },
            transformIndexHtml: {
                order: "pre",
                handler(html, { path: htmlPath }) {
                    const dom = new JSDOM(html);
                    const document = dom.window.document;
                    const scriptElems = document.getElementsByTagName("script");

                    for (const item of Array.from(scriptElems)) {
                        if (item.type === "module") continue;
                        if (item.hasAttribute("async")) continue;
                        if (item.hasAttribute("defer")) continue;

                        const src = item.getAttribute("src");

                        if (src) {
                            const absolutePath = path.join(__dirname, path.dirname(htmlPath), src);
                            const content = fs.readFileSync(absolutePath, "utf-8");

                            allScriptElemContent.push(content);
                        } else {
                            item.textContent && allScriptElemContent.push(item.textContent);
                        }

                        item.remove();
                    }

                    resolve(allScriptElemContent.join("\n"));
                    return dom.serialize();
                },
            },
            generateBundle(_, bundle) {
                const chunk = Object.values(bundle).find(
                    (chunk) =>
                        chunk.type === "chunk" && chunk.facadeModuleId === resolvedVirtualFileId,
                );

                if (!chunk) return;

                allScriptElemCode = chunk.code;
                delete bundle[chunk.fileName];
            },
        },
        {
            name: "vite-plugin-sync-script-post",
            apply: "build",
            transformIndexHtml: {
                order: "post",
                handler(html) {
                    const dom = new JSDOM(html);
                    const document = dom.window.document;
                    const scriptElem = document.createElement("script");

                    scriptElem.textContent = allScriptElemCode;
                    document.head.append(scriptElem);

                    return dom.serialize();
                },
            },
        },
    ];
}
```
