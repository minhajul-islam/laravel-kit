<template>
  <div class="flex-1">
    <div class="flex justify-between">
      <h1 class="font-mono text-xl">{{ command.name }}</h1>
      <kit-button @click.native="getOutputAsync">Run</kit-button>
    </div>
    <p class="mt-5 mb-3 text-base">{{ command.description }}</p>
    <div class="h-px bg-gray-300"></div>
    <div v-if="argumentsInit.length > 0">
      <h2 class="text-gray-800 font-bold text-base mt-4 dark:text-gray-200">Arguments</h2>
      <argument-input v-for="argument in argumentsInit" :key="argument.name" :field="argument" v-model="argument.value"></argument-input>
    </div>
    <div v-if="optionsInit.length > 0">
      <h2 class="text-gray-800 font-bold text-base mt-4 dark:text-gray-200">Options</h2>
      <option-input v-for="option in optionsInit" :key="option.name" :field="option" v-model="option.value"></option-input>
    </div>
    <div>
      <h2 class="text-gray-800 font-bold text-base mt-4 dark:text-gray-200">Terminal</h2>
      <div ref="terminal" class="font-mono text-sm mt-3 flex flex-col">
        <div class="select-text">
          <span>→</span> <span class="text-terminal-cyan dark:text-terminal-d-cyan">{{ appName }}</span> <span class="text-terminal-purple dark:text-terminal-d-purple">›</span>
          <span class="text-terminal-green dark:text-terminal-d-green">php</span>
          <span>artisan {{ fullCommand }}</span>
        </div>
        <pre class="max-h-64 overflow-y-auto pb-4 select-text overflow-x-auto whitespace-pre-wrap">
<code v-html="output"></code>
        </pre>
      </div>
      <div ref="terminal-end"></div>
    </div>
  </div>
</template>
<script>
import KitButton from "@/components/KitButton";
import ArgumentInput from "@/components/ArgumentInput";
import OptionInput from "@/components/OptionInput";
import { mapState } from "vuex";
import Anser from "anser";
import { exec } from "child_process";
import { remote } from "electron";
const { dialog } = remote;

export default {
  name: "Command",
  components: { KitButton, ArgumentInput, OptionInput },
  props: ["name"],
  data() {
    return {
      optionsInit: {},
      argumentsInit: {},
      output: ""
    };
  },
  computed: {
    ...mapState({ appName: "name" }),
    getName() {
      return this.name;
    },
    command() {
      return this.$store.state.project.commands.find((command) => command.name == this.name);
    },
    fullCommand() {
      let argumentsCommand = "",
        optionsCommand = "";
      if (this.argumentsInit.length > 0) {
        argumentsCommand = this.argumentsInit.reduce((tempCommand, argument) => {
          if (argument.value == "") {
            return `${tempCommand}`;
          }
          return `${tempCommand} ${argument.value}`;
        }, "");
      }
      if (this.optionsInit.length > 0) {
        optionsCommand = this.optionsInit.reduce((tempCommand, option) => {
          if (option.value == "" || option.value == false) {
            return `${tempCommand}`;
          }
          return `${tempCommand} ${option.name}${option.accept_value ? " " + option.value : ""}`;
        }, "");
      }
      let globalCommand = "";
      if (this.$store.state.env != "") {
        globalCommand += ` --env ${this.$store.state.env}`;
      }
      if (this.$store.state.verbosity != 1) {
        globalCommand += ` -${"v".repeat(this.$store.state.verbosity)}`;
      }
      return this.command.name + argumentsCommand + optionsCommand + globalCommand;
    }
  },
  methods: {
    getArguments() {
      this.argumentsInit = Object.keys(this.command.definition.arguments)
        .map((argument) => this.command.definition.arguments[argument])
        .map((option) => Object.assign({}, option, { value: "" }));
    },
    getOptions() {
      const remove = ["--help", "--quiet", "--verbose", "--version", "--ansi", "--no-ansi", "--no-interaction", "--env"];
      this.optionsInit = Object.keys(this.command.definition.options)
        .map((option) => this.command.definition.options[option])
        .filter((option) => !remove.includes(option.name))
        .map((option) => Object.assign({}, option, { value: option.accept_value ? "" : option.default }));
    },
    getOutputAsync() {
      this.output = "Running...";
      this.$store.state.running = true;
      exec(`php artisan ${this.fullCommand} --no-interaction --ansi`, { cwd: this.$store.state.dir }, (error, stdout) => {
        if (error) {
          if (stdout.includes("Could not open input file: artisan")) {
            let message = `${this.$store.state.dir} - This folder is not a Laravel project. Please create a Laravel project and then open it.`;
            dialog.showMessageBox({
              type: "error",
              title: "Error",
              message
            });
          }
        }
        this.output = Anser.ansiToHtml(stdout, { use_classes: true });
        this.$refs["terminal-end"].scrollIntoView();
        this.$store.state.running = false;
      });
      this.$refs["terminal-end"].scrollIntoView();
    }
  },
  mounted() {
    this.getArguments();
    this.getOptions();
  }
};
</script>

<style>
/* styles in styles.css */
</style>
