<!--
Copyright 2024 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <li>
    <v-dialog v-model="showTaskConfigForm" width="600">
      <v-card width="600" class="mx-auto">
        <v-card-title>Config for {{ node.display_name }}</v-card-title>
        <v-card-text>
          <task-config-form
            :fields="node.task_config"
            @save="saveTaskConfig($event)"
            @cancel="showTaskConfigForm = false"
          ></task-config-form>
        </v-card-text>
      </v-card>
    </v-dialog>
    <span
      class="node-content"
      :class="$vuetify.theme.name === 'dark' ? '' : 'white-background'"
    >
      <v-menu
        v-if="celeryTask.status_short"
        activator="parent"
        location="bottom"
        :close-on-content-click="false"
      >
        <v-card width="600px" class="pa-4 mt-2">
          <task-result-default
            :worker="workflow"
            :task="celeryTask"
          ></task-result-default>
        </v-card>
      </v-menu>
      <template v-if="node.isRoot">
        <div
          v-for="file in workflow.files"
          :key="file.id"
          :class="file.is_deleted ? 'red-text' : ''"
          style="font-size: 0.9em"
        >
          <v-icon v-if="file.data_type.startsWith('cloud:')" class="mr-2 mt-n1"
            >mdi-cloud-outline</v-icon
          >
          <v-icon v-else size="small" class="mr-1" style="margin-top: -3px"
            >mdi-file-outline</v-icon
          >
          {{ file.display_name }}
        </div>
      </template>
      <template v-else>
        <div>
          <task-status-icon
            class="mr-1"
            :task-status="celeryTask.status_short"
          ></task-status-icon>
          {{ node.display_name }}
          <span v-if="celeryTask.runtime">
            <br /><small class="ml-1"
              >Runtime:
              <strong
                >{{ celeryTask.runtime.toFixed(1) }} seconds</strong
              ></small
            >
          </span>

          <v-icon
            v-if="!workflow.tasks.length"
            size="small"
            color="grey-lighten-1"
            >mdi-plus</v-icon
          >
        </div>

        <div v-if="hasTaskConfig" class="mt-2 pl-1">
          <div v-for="option in node.task_config">
            <small
              v-if="
                option.hasOwnProperty('value') &&
                option.value !== null &&
                option.value !== undefined &&
                option.value !== ''
              "
            >
              {{ option.name }}:
              {{ option.value }}
            </small>
          </div>
          <v-btn
            v-if="!Object.keys(celeryTask).length"
            block
            variant="tonal"
            size="small"
            class="text-none custom-border-color mt-2"
            @click.stop="showTaskConfigForm = true"
          >
            <v-icon left class="mr-1">mdi-cog-outline</v-icon>
            Configure
          </v-btn>
        </div>
      </template>

      <workflow-task-dropdown
        v-if="!workflow.tasks.length"
        :isRootNode="node.isRoot"
        @add-task="addTask($event, node)"
        @remove-task="removeTask(node)"
      ></workflow-task-dropdown>
    </span>
    <ul v-if="node.tasks && node.tasks.length">
      <tree-node
        v-for="task in node.tasks"
        :node="task"
        :workflow="workflow"
        :add-task="addTask"
        :remove-task="removeTask"
        :update-workflow="updateWorkflow"
      ></tree-node>
    </ul>
  </li>
</template>

<script>
import WorkflowTaskDropdown from "./WorkflowTaskDropdown";
import TaskStatusIcon from "./TaskStatusIcon";
import TaskResultDefault from "@/components/TaskResultDefault.vue";
import TaskConfigForm from "./TaskConfigForm.vue";

export default {
  name: "treeNode",
  props: {
    node: Object,
    workflow: Object,
    addTask: Function,
    removeTask: Function,
    updateWorkflow: Function,
  },
  components: {
    WorkflowTaskDropdown,
    TaskStatusIcon,
    TaskResultDefault,
    TaskConfigForm: TaskConfigForm,
  },
  data() {
    return {
      showTaskConfigForm: false,
    };
  },
  computed: {
    celeryTask() {
      return (
        this.workflow.tasks.filter((task) => task.uuid === this.node.uuid)[0] ||
        {}
      );
    },
    hasTaskConfig() {
      return this.node.task_config && this.node.task_config.length;
    },
    hasTaskConfigWithValue() {
      return this.node.task_config.some((option) => {
        return (
          option.hasOwnProperty("value") &&
          option.value !== null &&
          option.value !== undefined &&
          option.value !== ""
        );
      });
    },
    nodeIcon() {
      return this.celeryTask.status_short === "PROGRESS"
        ? "mdi-chart-box-outline"
        : "mdi-information-outline";
    },
  },
  methods: {
    saveTaskConfig(formData) {
      // Loop through the task options in the object
      this.node.task_config.forEach((option) => {
        // Check if there's a corresponding value in the formData
        if (formData.hasOwnProperty(option.name)) {
          // Update the option with the value from the formData
          option.value = formData[option.name];
        }
      });
      this.updateWorkflow();
      this.showTaskConfigForm = false;
    },
  },
};
</script>

<style scoped>
.white-background {
  background-color: white;
}
.red-text {
  color: red;
}
</style>
