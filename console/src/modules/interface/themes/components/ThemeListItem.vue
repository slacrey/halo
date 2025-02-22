<script lang="ts" setup>
import {
  VTag,
  VStatusDot,
  Toast,
  VDropdownItem,
  VDropdown,
  VDropdownDivider,
  VButton,
  VSpace,
  IconMore,
} from "@halo-dev/components";
import type { Theme } from "@halo-dev/api-client";
import { apiClient } from "@/utils/api-client";
import { toRefs, ref, inject, type Ref } from "vue";
import { useThemeLifeCycle } from "../composables/use-theme";
import { usePermission } from "@/utils/permission";
import { useI18n } from "vue-i18n";
import { useQueryClient } from "@tanstack/vue-query";
import { useEntityDropdownItemExtensionPoint } from "@/composables/use-entity-extension-points";
import { markRaw } from "vue";
import { defineComponent } from "vue";
import UninstallOperationItem from "./operation/UninstallOperationItem.vue";
import { computed } from "vue";

const { currentUserHasPermission } = usePermission();
const { t } = useI18n();
const queryClient = useQueryClient();

const props = withDefaults(
  defineProps<{
    theme: Theme;
    installed?: boolean;
    isSelected?: boolean;
  }>(),
  {
    installed: true,
    isSelected: false,
  }
);

const emit = defineEmits<{
  (event: "upgrade"): void;
  (event: "preview"): void;
  (event: "select", theme: Theme): void;
}>();

const { theme } = toRefs(props);

const activeTabId = inject<Ref<string>>("activeTabId", ref(""));

const {
  isActivated,
  getFailedMessage,
  handleActiveTheme,
  handleResetSettingConfig,
} = useThemeLifeCycle(theme);

// Creating theme
const creating = ref(false);

const handleCreateTheme = async () => {
  try {
    creating.value = true;

    const { data } =
      await apiClient.extension.theme.createthemeHaloRunV1alpha1Theme({
        theme: props.theme,
      });

    // create theme settings
    apiClient.theme.reload({ name: data.metadata.name });

    activeTabId.value = "installed";

    Toast.success(t("core.common.toast.install_success"));
  } catch (error) {
    console.error("Failed to create theme", error);
  } finally {
    creating.value = false;
    queryClient.invalidateQueries({ queryKey: ["installed-themes"] });
    queryClient.invalidateQueries({ queryKey: ["not-installed-themes"] });
  }
};

const { dropdownItems } = useEntityDropdownItemExtensionPoint<Theme>(
  "theme:list-item:operation:create",
  theme,
  computed(() => [
    {
      priority: 10,
      component: markRaw(VButton),
      props: {
        size: "sm",
      },
      action: () => handleActiveTheme(true),
      label: t("core.common.buttons.activate"),
      visible:
        !isActivated.value &&
        currentUserHasPermission(["system:themes:manage"]),
    },
    {
      priority: 20,
      component: markRaw(VButton),
      props: {
        size: "sm",
      },
      action: () => {
        emit("select", props.theme);
      },
      label: t("core.common.buttons.select"),
    },
    {
      priority: 30,
      component: markRaw(
        defineComponent({
          components: {
            VButton,
            IconMore,
          },
          template: `<VButton size="sm"><IconMore /></VButton>`,
        })
      ),
      visible: currentUserHasPermission(["system:themes:manage"]),
      children: [
        {
          priority: 10,
          component: markRaw(VDropdownItem),
          action: () => {
            emit("preview");
          },
          label: t("core.common.buttons.preview"),
        },
        {
          priority: 20,
          component: markRaw(VDropdownDivider),
        },
        {
          priority: 30,
          component: markRaw(UninstallOperationItem),
          props: {
            theme: props.theme,
          },
        },
        {
          priority: 40,
          component: markRaw(VDropdownItem),
          props: {
            type: "danger",
          },
          action: () => handleResetSettingConfig(),
          label: t("core.common.buttons.reset"),
        },
      ],
    },
  ])
);
</script>

<template>
  <div
    class="group relative flex grid-cols-1 flex-col overflow-hidden rounded bg-white p-0 shadow-sm transition-all duration-500 hover:shadow-md hover:ring-1 sm:grid sm:grid-cols-7 sm:p-2"
    :class="{ 'ring-1': isSelected }"
  >
    <div class="col-span-2">
      <div class="relative block">
        <div class="aspect-h-9 aspect-w-16">
          <div
            class="transform-gpu rounded-none bg-cover bg-center bg-no-repeat sm:rounded"
            :style="{
              backgroundImage: `url(${theme.spec.logo})`,
            }"
          >
            <div
              class="flex h-full w-full items-center justify-center rounded-none backdrop-blur-3xl sm:rounded"
            >
              <img class="h-16 w-16 rounded" :src="theme.spec.logo" />
            </div>
          </div>
        </div>
      </div>
    </div>
    <div
      class="relative col-span-5 grid grid-cols-1 content-between p-2 sm:px-4 sm:py-1"
    >
      <div>
        <div class="flex flex-wrap items-center justify-between gap-2">
          <div class="inline-flex items-center gap-2">
            <div
              class="relative block cursor-pointer text-sm font-medium text-black transition-all hover:text-gray-600 hover:underline sm:text-base"
              @click="emit('select', theme)"
            >
              {{ theme.spec.displayName }}
            </div>
            <span class="text-xs text-gray-500 sm:text-sm">
              {{ theme.spec.version }}
            </span>
            <VTag v-if="isActivated" theme="primary">
              {{ $t("core.common.status.activated") }}
            </VTag>
          </div>
          <div>
            <VStatusDot
              v-if="getFailedMessage()"
              v-tooltip="getFailedMessage()"
              state="warning"
              animate
            />
            <VStatusDot
              v-if="theme.metadata.deletionTimestamp"
              v-tooltip="$t('core.common.status.deleting')"
              state="warning"
              animate
            />
          </div>
        </div>
        <p
          class="mt-2 line-clamp-1 text-xs font-normal text-gray-500 sm:text-sm"
        >
          {{ theme.spec.description }}
        </p>
      </div>
      <div
        class="mt-4 flex w-full flex-1 items-center justify-between gap-2 sm:mt-0"
      >
        <div v-if="theme.spec.author" class="inline-flex items-center gap-1.5">
          <a
            v-if="theme.spec.author.website"
            class="text-xs text-gray-700 hover:text-gray-900"
            :href="theme.spec.author.website"
            target="_blank"
          >
            {{ theme.spec.author.name }}
          </a>
          <span v-else class="text-xs text-gray-700">
            {{ theme.spec.author.name }}
          </span>
        </div>
        <div>
          <VSpace v-if="installed">
            <template v-for="(item, index) in dropdownItems" :key="index">
              <template v-if="!item.children?.length">
                <component
                  :is="item.component"
                  v-if="item.visible !== false"
                  v-permission="item.permissions"
                  v-bind="item.props"
                  @click="item.action?.(theme)"
                >
                  {{ item.label }}
                </component>
              </template>
              <template v-else>
                <VDropdown
                  v-if="item.visible !== false"
                  v-permission="item.permissions"
                >
                  <component
                    :is="item.component"
                    v-bind="item.props"
                    @click="item.action?.(theme)"
                  >
                    {{ item.label }}
                  </component>
                  <template #popper>
                    <template
                      v-for="(childItem, childIndex) in item.children"
                      :key="`child-${childIndex}`"
                    >
                      <component
                        :is="childItem.component"
                        v-if="childItem.visible !== false"
                        v-permission="childItem.permissions"
                        v-bind="childItem.props"
                        @click="childItem.action?.(theme)"
                      >
                        {{ childItem.label }}
                      </component>
                    </template>
                  </template>
                </VDropdown>
              </template>
            </template>
          </VSpace>
          <VButton
            v-if="
              !installed && currentUserHasPermission(['system:themes:manage'])
            "
            size="sm"
            :loading="creating"
            @click="handleCreateTheme"
          >
            {{ $t("core.common.buttons.install") }}
          </VButton>
        </div>
      </div>
    </div>
  </div>
</template>
