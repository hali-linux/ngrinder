<template>
    <div class="row d-flex" id="running-container">
        <div class="intro left-panel"
             :data-step="shownBsTab ? 4 : undefined"
             :data-intro="shownBsTab ? i18n('intro.running.summary') : undefined">
            <fieldSet>
                <legend class="border-bottom" v-text="i18n('perfTest.running.summaryTitle')"></legend>
            </fieldSet>
            <div class="d-flex">
                <fieldset class="w-100">
                    <div class="my-4">
                        <label v-text="i18n('perfTest.running.totalVusers')"></label>
                        <strong v-text="test.vuserPerAgent * test.agentCount"></strong>
                        <span class="badge badge-info pull-right">
                            <span v-text="i18n('perfTest.running.running')"></span>
                            <span v-text="runningThread"></span>
                        </span>
                    </div>
                    <div class="my-4">
                        <label v-text="i18n('perfTest.running.totalProcesses')"></label>
                        <span v-text="test.processes * test.agentCount"></span>
                        <span class="badge badge-info float-right">
                            <span v-text="i18n('perfTest.running.running')"></span>
                            <span v-text="runningProcess"></span>
                        </span>
                    </div>
                    <hr>
                    <div class="my-4">
                        <label class="mb-1 align-top" v-text="i18n('perfTest.config.targetHost')"></label>
                        <div class="d-inline-block">
                            <div v-for="host in test.targetHosts.split(',')" v-text="host.trim()"></div>
                        </div>
                    </div>
                    <hr>
                    <div v-if="test.threshold === 'D'" class="my-4">
                        <label v-text="i18n('perfTest.running.duration')"></label>
                        <span v-text="test.duration" class="mr-2"></span>
                        <code>HH:MM:SS</code>
                        <div class="badge badge-success float-right">
                            <span v-text="i18n('perfTest.running.runCount')"></span>
                            <span>{{ (totalStatistics.Tests + totalStatistics.Errors) | numFormat }}</span>
                        </div>
                    </div>
                    <div v-else class="my-4">
                        <label v-text="i18n('perfTest.running.totalRunCount')"></label>
                        <span v-text="test.runCount * test.agentCount * test.vuserPerAgent"></span>
                        <div class="badge badge-success float-right">
                            <span v-text="i18n('perfTest.running.runCount')"></span>
                            <span>{{ (totalStatistics.Tests + totalStatistics.Errors) | numFormat }}</span>
                        </div>
                    </div>
                    <div>
                        <label v-text="i18n('perfTest.running.targetState')"></label>
                    </div>
                    <div class="monitor-state my-4">
                        <ul v-for="(monitor, name) in monitorState">
                            <li>
                                <div class="ellipsis">
                                    <span><b v-text="getShortenString(name)"></b></span>
                                    <span v-text="getPackageState(monitor)"></span>
                                </div>
                            </li>
                        </ul>
                    </div>
                    <div>
                        <label v-text="i18n('perfTest.running.agentState')"></label>
                    </div>
                    <div class="agent-state my-4">
                        <ul v-for="(agent, name) in agentState">
                            <li>
                                <div class="ellipsis">
                                    <span><b v-text="getShortenString(name)"></b></span>
                                    <span v-text="getPackageState(agent)"></span>
                                </div>
                            </li>
                        </ul>
                    </div>
                </fieldset>
            </div>
        </div>
        <!-- end running content left -->
        <div class="ml-auto right-panel">
            <fieldSet class="mb-3">
                <legend class="border-bottom">
                    <span v-text="i18n('perfTest.running.tpsGraph')"></span>
                    <span class="badge badge-success" v-text="formatTestTime(testTime)"></span>
                    <button @click.prevent="stopRunningTest" class="btn btn-danger float-right intro"
                            :data-step="shownBsTab ? 5 : undefined"
                            :data-intro="shownBsTab ? i18n('intro.running.stopButton') : undefined">
                        <i class="fa fa-stop mr-1"></i>
                        <span v-text="i18n('common.button.stop')"></span>
                    </button>
                </legend>
            </fieldSet>
            <div id="running-tps-chart" class="chart w-100"></div>
            <div class="intro mt-1"
                 :data-step="shownBsTab ? 6 : undefined"
                 :data-intro="shownBsTab ? i18n('intro.running.accumulated') : undefined">
                <ul class="nav nav-tabs border-bottom-0">
                    <li class="nav-item">
                        <a href="#last-sample-tab" data-toggle="tab" class="nav-link active"
                           v-text="i18n('perfTest.running.latestSample')"></a>
                    </li>
                    <li class="nav-item">
                        <a href="#accumulated-sample-tab" data-toggle="tab" class="nav-link"
                           v-text="i18n('perfTest.running.accumulatedStatistic')"></a>
                    </li>
                </ul>
                <div class="tab-content">
                    <div class="tab-pane active" id="last-sample-tab">
                        <sampling-table :statistics="lastSampleStatistics"></sampling-table>
                    </div>
                    <div class="tab-pane" id="accumulated-sample-tab">
                        <sampling-table :type="'accumulated'" :statistics="cumulativeStatistics"></sampling-table>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import { Component, Prop } from 'vue-property-decorator';
    import { Mixins } from 'vue-mixin-decorator';
    import Base from '../../Base.vue';
    import ControlGroup from '../../common/ControlGroup.vue';
    import MessagesMixin from '../../common/mixin/MessagesMixin.vue';
    import SamplingTable from './SamplingTable.vue';
    import Chart from '../../../chart.js';
    import Queue from '../../../queue.js';
    import FormatMixin from '../../common/mixin/FormatMixin.vue';

    @Component({
        name: 'running',
        components: { ControlGroup, SamplingTable },
    })
    export default class Running extends Mixins(Base, FormatMixin, MessagesMixin) {
        @Prop({ type: Object, required: true })
        testProp;

        lastSampleStatistics = [];
        cumulativeStatistics = [];
        totalStatistics = { Tests: 0, Errors: 0 };
        agentState = {};
        monitorState = {};
        samplingIntervalId = -1;
        runningProcess = 0;
        runningThread = 0;
        testTime = 0;

        tpsQueue = {};
        tpsChart = {};

        test = {};
        shownBsTab = false;

        created() {
            Object.assign(this.test, this.testProp);
            this.tpsQueue = new Queue(60 / this.test.samplingInterval);
            this.tpsChart = new Chart('running-tps-chart', [this.tpsQueue.getArray()], this.test.samplingInterval);
        }

        startSamplingInterval() {
            this.updateSamplingData();
            this.samplingIntervalId = setInterval(this.updateSamplingData, 1000 * this.test.samplingInterval);
        }

        updateSamplingData() {
            if (!this.test.id) {
                return;
            }
            this.$http.get(`/perftest/api/${this.test.id}/sample`).then(res => {
                const perfTestSample = res.data.perf;
                if (perfTestSample) {
                    this.lastSampleStatistics = perfTestSample.lastSampleStatistics;
                    this.cumulativeStatistics = perfTestSample.cumulativeStatistics;
                    this.totalStatistics = perfTestSample.totalStatistics;
                    this.runningProcess = perfTestSample.process;
                    this.runningThread = perfTestSample.thread;
                    this.testTime = perfTestSample.testTime;
                    this.tpsQueue.enQueue(perfTestSample.tpsChartData);
                    if (this.shownBsTab) {
                        this.tpsChart.plot();
                    }
                }
                this.agentState = res.data.agent || {};
                this.monitorState = res.data.monitor || {};
            });
        }

        stopRunningTest() {
            this.$bootbox.confirm({
                message: this.i18n('perfTest.message.stop.confirm'),
                buttons: {
                    confirm: { label: this.i18n('common.button.ok') },
                    cancel: { label: this.i18n('common.button.cancel') },
                },
                onConfirm: () => this.$http.put(`/perftest/api/${this.test.id}?action=stop`)
                    .then(() => this.showSuccessMsg(this.i18n('perfTest.message.stop.success')))
                    .catch(() => this.showErrorMsg(this.i18n('perfTest.message.stop.error'))),
            });
        }

        // monitor or agent state
        getPackageState(targetPackage) {
            let packageState = `CPU-${this.formatPercentage(null, targetPackage.cpuUsedPercentage)}
            MEM-${this.formatPercentage(null, ((targetPackage.totalMemory - targetPackage.freeMemory) / targetPackage.totalMemory) * 100)}`;

            if (targetPackage.receivedPerSec !== 0 || targetPackage.sentPerSec !== 0) {
                packageState += ` RX-${this.formatNetwork(null, targetPackage.receivedPerSec)} TX-${this.formatNetwork(null, targetPackage.sentPerSec)}`;
            }
            return packageState;
        }

        getShortenString(str, start, end) {
            start = start || 0;
            end = end || 20;
            if (str.length >= end) {
                str = str.substr(start, end - 4);
                str += '...';
            }
            return str;
        }
    }
</script>

<style lang="less">
    #running-container {
        .control-label {
            width: 170px;
        }
    }
</style>

<style lang="less" scoped>
    #running-container {
        #running-tps-chart {
            height: 300px;
        }

        .left-panel {
            width: 380px;
        }

        .right-panel {
            width: 540px;
        }

        .agent-state, .monitor-state {
            margin-left: -20px;

            ul {
                list-style: none;

                li {
                    height: 20px;

                    div {
                        width: 100%;
                    }
                }
            }
        }

        .badge {
            font-size: 11px;
            padding: 3px 6px;
        }

        label {
            width: 110px;
            color: #666;
            font-weight: bold;
        }

        legend, label {
            margin-bottom: 0;
        }

        strong {
            color: #6DAFCF;
        }

        .chart {
            width: 610px;
            height: 300px;
            border: 1px solid #666;
            margin-bottom: 12px;
        }
    }
</style>
