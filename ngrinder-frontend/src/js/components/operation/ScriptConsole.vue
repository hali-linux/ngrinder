<template>
    <div class="container">
        <vue-headful :title="i18n('operation.script.title')"/>
        <fieldset>
            <legend class="header border-bottom d-flex">
                <span v-text="i18n('navigator.dropDown.scriptConsole')"></span>
                <button class="btn btn-success mt-auto mb-auto ml-auto" @click="runScript">
                    <i class="fa fa-play mr-1"></i>
                    <span v-text="i18n('operation.script.runScript')"></span>
                </button>
            </legend>
        </fieldset>
        <code-mirror ref="editor"></code-mirror>
        <pre class="rounded border" v-text="result"></pre>
    </div>
</template>

<script>
    import Component from 'vue-class-component';
    import VueHeadful from 'vue-headful';

    import Base from '../Base.vue';
    import CodeMirror from '../common/CodeMirror.vue';

    @Component({
        name: 'scriptConsole',
        components: { CodeMirror, VueHeadful },
    })
    export default class ScriptConsole extends Base {
        result = 'You can write groovy code to monitor the ngrinder internal state.\n' +
            '\n' +
            'Following variables are available.\n' +
            '\n' +
            '- applicationContext  (org.springframework.context.ApplicationContext)\n' +
            '- agentManager        (org.ngrinder.perftest.service.AgentManager)\n' +
            '- agentManagerService (org.ngrinder.agent.service.AgentManagerService)\n' +
            '- regionService       (org.ngrinder.region.service.RegionService)\n' +
            '- consoleManager      (org.ngrinder.perftest.service.ConsoleManager)\n' +
            '- userService         (org.ngrinder.user.service.UserService)\n' +
            '- perfTestService     (org.ngrinder.perftest.service.PerfTestService)\n' +
            '- tagService          (org.ngrinder.perftest.service.TagService)\n' +
            '- fileEntryService    (org.ngrinder.script.service.FileEntryService)\n' +
            '- config              (org.ngrinder.infra.config.Config)\n' +
            '- pluginManager       (org.ngrinder.infra.plugin.PluginManager)\n' +
            '- cacheManager        (org.springframework.cache.CacheManager)\n' +
            '\n' +
            'Please type following and click the Submit button as a example\n' +
            '\n' +
            'print agentManager.getAllAttachedAgents()\n' +
            '\n' +
            'please refer nGrinder javadoc to find out more APIs on the given variables.\n';

        mounted() {
            this.codemirror.setSize(null, 400);
        }

        get codemirror() {
            return this.$refs.editor.codemirror;
        }

        runScript() {
            const script = this.codemirror.getValue().trim();
            if (!script) {
                return;
            }

            this.$http.post('/operation/script_console/api', {
                script,
            })
            .then(res => this.result = res.data.result);
        }
    }
</script>
<style lang="less" scoped>

    pre {
        height: 250px;
        margin: 5px 0 20px;
        padding: 14px 10px;
        max-height: 340px;
        overflow-y: scroll;
        font-size: 12px;
        background-color: #f5f5f5;
    }

    button {
        height: 30px;
    }
</style>
