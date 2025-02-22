# Entity 数据列表操作菜单扩展点

## 原由

目前 Halo 2 的 Console 中，展示数据列表是统一使用 Entity 组件，Entity 组件中提供了用于放置操作按钮的插槽，此扩展点用于支持通过插件扩展部分数据列表的操作菜单项。

## 定义方式

目前支持扩展的数据列表：

- 文章：`"post:list-item:operation:create"?: (post: Ref<ListedPost>) => | EntityDropdownItem<ListedPost>[] | Promise<EntityDropdownItem<ListedPost>[]>`
- 插件：`"plugin:list-item:operation:create"?: (plugin: Ref<Plugin>) => | EntityDropdownItem<Plugin>[] | Promise<EntityDropdownItem<Plugin>[]>`
- 备份：`"backup:list-item:operation:create"?: (backup: Ref<Backup>) => | EntityDropdownItem<Backup>[] | Promise<EntityDropdownItem<Backup>[]>`
- 主题：`"theme:list-item:operation:create"?: (theme: Ref<Theme>) => | EntityDropdownItem<Theme>[] | Promise<EntityDropdownItem<Theme>[]>`

示例：

> 此示例是在文章列表中添加一个`导出为 Markdown 文档`的操作菜单项。

```ts
import type { ListedPost } from "@halo-dev/api-client";
import { VDropdownItem } from "@halo-dev/components";
import { definePlugin } from "@halo-dev/console-shared";
import axios from "axios";
import { markRaw } from "vue";

export default definePlugin({
  extensionPoints: {
    "post:list-item:operation:create": () => {
      return [
        {
          priority: 21,
          component: markRaw(VDropdownItem),
          label: "导出为 Markdown 文档",
          visible: true,
          permissions: [],
          action: async (post: ListedPost) => {
            const { data } = await axios.get(
              `/apis/api.console.halo.run/v1alpha1/posts/${post.post.metadata.name}/head-content`
            );
            const blob = new Blob([data.raw], {
              type: "text/plain;charset=utf-8",
            });
            const url = window.URL.createObjectURL(blob);
            const link = document.createElement("a");
            link.href = url;
            link.download = `${post.post.spec.title}.md`;
            link.click();
          },
        },
      ];
    },
  },
});
```

`EntityDropdownItem` 类型：

```ts
export interface EntityDropdownItem<T> {
  priority: number;                     // 优先级，越小越靠前
  component: Raw<Component>;            // 菜单项组件，可以使用 `@halo-dev/components` 中提供的 `VDropdownItem`，也可以自定义
  props?: Record<string, unknown>;      // 组件的 props
  action?: (item?: T) => void;          // 点击事件
  label?: string;                       // 菜单项名称
  visible?: boolean;                    // 是否可见
  permissions?: string[];               // 权限
  children?: EntityDropdownItem<T>[];   // 子菜单
}
```
