## Description

Collect the content of all blocking `<script>` tags in `index.html`, then perform bundling according to the Vite configuration file, and finally fill the bundling results back into `index.html`.

It's particularly useful if you need to insert blocking scripts into `index.html` and want these scripts to benefit from Vite's bundling process.

> ⚠️ Work in progress...

## Implementation
```ts
import type { Plugin } from "vite";
import path from "node:path";
import crypto from "node:crypto";
import fs from "node:fs";
import { JSDOM } from "jsdom";

export default function extraEntryPlugin(): Plugin {
    const allScriptElemContent: string[] = [];
    const virtualFileId = `virtual:${crypto.randomUUID()}`;
    const resolvedVirtualFileId = `\0${virtualFileId}`;
    const { promise, resolve } = Promise.withResolvers();

    let allScriptElemCode = "";

    return [
        {
            name: "vite-plugin-inject-sync-script-pre",
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
                            // 处理没有配置 rollupOptions 的情况
                            ...(!config.build?.rollupOptions?.input ? { index: "index.html" } : {}),
                            ...config.build?.rollupOptions?.input,
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

                    const filteredScriptElems = Array.from(scriptElems).filter((item) => {
                        if (item.type === "module") return false;
                        if (item.hasAttribute("async")) return false;
                        if (item.hasAttribute("defer")) return false;

                        return true;
                    });

                    let count = 0;

                    for (const item of Array.from(scriptElems)) {
                        if (item.type === "module") continue;
                        if (item.hasAttribute("async")) continue;
                        if (item.hasAttribute("defer")) continue;

                        const src = item.getAttribute("src");

                        count++;

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
                // 找到虚拟文件的打包结果
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
            name: "vite-plugin-inject-sync-script-post",
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
