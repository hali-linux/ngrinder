<template>
    <div class="container perftest-list-container p-0">
        <vue-headful :title="i18n('perfTest.list.title')"></vue-headful>
        <div class="img-unit" :style="`background-image: url('${contextPath}/img/bg_perftest_banner_en.png')`"></div>
        <div class="current-running-status-container">
            <code v-text="runningSummary"></code>
        </div>

        <search-bar ref="searchBar" @filter-running="runQueryFilter" @filter-schduled="runQueryFilter"
                    @create="$router.push('/perftest/new')"
                    @search="updateTableWithUrl" @change-tag="updateTableWithUrl"
                    @delete-selected-tests="deleteTests($refs.vuetable.selectedTo.join(','))"></search-bar>
        <vuetable
            v-show="showTable"
            ref="vuetable"
            data-path="tests"
            pagination-path=""
            detail-row-component="small-chart"
            :api-url="`${contextPath}/perftest/api/list`"
            :append-params="table.appendParams"
            :query-params="{ sort: 'sort', page: 'page.page', perPage: 'page.size' }"
            :per-page="table.pagination.perPage"
            :css="table.css.table"
            :multi-sort="true"
            :fields="tableFields"
            :load-on-start="false"
            @vuetable:pagination-data="beforePagination"
            @vuetable:loading="beforeTableLoading"
            @vuetable:loaded="tableLoaded">

            <template slot="status" slot-scope="props">
                <div class="ball"
                   data-toggle="popover"
                   data-html="true"
                   data-trigger="hover"
                   :id="`ball_${props.rowData.id}`"
                   :title="props.rowData.status.name"
                   :data-content="`${props.rowData.progressMessage}<br><b>${props.rowData.lastProgressMessage}</b>`.replace(/\n/g, '<br>')">
                    <img :src="`${contextPath}/img/ball/${props.rowData.status.iconName}`">
                </div>
            </template>

            <template slot="testName" slot-scope="props">
                <div class="ellipsis testName"
                     data-toggle="popover"
                     data-html="true"
                     data-trigger="hover"
                     :title="props.rowData.testName"
                     :data-content="getTestNamePopoverContent(props.rowData).replace(/\n/g, '<br>')">
                    <router-link :to="`/perftest/${props.rowData.id}`" target="_self" v-text="props.rowData.testName"></router-link>
                </div>
            </template>

            <template slot="scriptName" slot-scope="props">
                <div class="ellipsis scriptName"
                     data-toggle="popover"
                     data-html="true"
                     data-trigger="hover"
                     :title="i18n('perfTest.list.scriptName')"
                     :data-content="`${props.rowData.scriptName}<br> - ${i18n('script.list.revision')} : ${(props.rowData.scriptRevision)}`">
                    <a v-if="isAdmin"
                       :href="`/script/detail/${props.rowData.scriptName}?r=${(props.rowData.scriptRevision)}&ownerId=${(props.rowData.createdUser.userId)}`"
                       v-text="props.rowData.scriptName"></a>
                    <a v-else :href="`/script/detail/${props.rowData.scriptName}?r=${(props.rowData.scriptRevision)}`"
                       v-text="props.rowData.scriptName"></a>
                </div>
            </template>

            <template v-if="ngrinder.config.clustered" slot="region" slot-scope="props">
                <div class="ellipsis region" :title="props.rowData.region" v-text="props.rowData.region"></div>
            </template>

            <template slot="createUser" slot-scope="props">
                <div class="ellipsis createUser" :title="props.rowData.createdUser.userName" v-text="props.rowData.createdUser.userName"></div>
            </template>

            <template slot="threshold" slot-scope="props">
                <div v-if="props.rowData.threshold === 'D'" :title="i18n('perfTest.list.duration')" v-text="props.rowData.duration"></div>
                <div v-else v-text="props.rowData.runCount" :title="i18n('perfTest.list.runCount')"></div>
            </template>

            <template slot="startTime" slot-scope="props">
                <div class="startTime">
                    <div v-if="props.rowData.startTime">{{ props.rowData.startTime | dateFormat('YYYY-MM-DD HH:mm') }}</div>
                </div>
            </template>

            <template slot="tps" slot-scope="props">
                <div class="tps">
                    <div v-if="$utils.exists(props.rowData.tps)">{{ props.rowData.tps | numFormat('0,0.0') }}</div>
                </div>
            </template>

            <template slot="meanTestTime" slot-scope="props">
                <div class="meanTestTime">
                    <div v-if="$utils.exists(props.rowData.meanTestTime)">{{ props.rowData.meanTestTime | numFormat('0,0.0') }}</div>
                </div>
            </template>

            <template slot="errorRate" slot-scope="props">
                <div class="ellipsis errorRate"
                     data-toggle="popover"
                     data-html="true"
                     data-trigger="hover"
                     data-placement="top"
                     :data-content="getErrorRatePopoverContent(props.rowData)"
                     v-text="getErrorRate(props.rowData.tests, props.rowData.errors)">
                </div>
            </template>

            <template slot="vuser" slot-scope="props">
                <div class="ellipsis"
                     data-toggle="popover"
                     data-html="true"
                     data-trigger="hover"
                     data-placement="left"
                     :data-content="getVuserPopoverContent(props.rowData)">
                     {{ props.rowData.vuserPerAgent * props.rowData.agentCount | numFormat }}
                </div>
            </template>

            <template slot="actions" slot-scope="props">
                <i v-if="props.rowData.status.reportable" @click="showChart(props.rowData.id)"
                   :title="i18n('perfTest.action.showChart')" class="fa fa-line-chart pointer-cursor"></i>
                <i v-if="props.rowData.status.deletable" @click="deleteTests(props.rowData.id)"
                   :title="i18n('common.button.delete')" class="fa fa-remove pointer-cursor"></i>
                <i v-if="props.rowData.status.stoppable" @click="stopTest(props.rowData.id)"
                   :title="i18n('common.button.stop')" class="fa fa-stop pointer-cursor"></i>
            </template>

        </vuetable>
        <vuetable-pagination
            ref="pagination"
            :css="table.css.pagination"
            @vuetable-pagination:change-page="changePage">
        </vuetable-pagination>
    </div>
</template>

<script>
    import { Mixins } from 'vue-mixin-decorator';
    import { Component } from 'vue-property-decorator';
    import Vuetable from 'vuetable-2';
    import VuetablePagination from 'vuetable-2/src/components/VuetablePagination.vue';
    import vueHeadful from 'vue-headful';
    import Vue from 'vue';

    import Base from '../../Base.vue';
    import SearchBar from './Searchbar.vue';
    import IntroButton from '../../common/IntroButton.vue';
    import MessagesMixin from '../../common/mixin/MessagesMixin.vue';
    import SmallChart from './SmallChart.vue';
    import TableConfig from './mixin/TableConfig.vue';

    Vue.component('small-chart', SmallChart);

    @Component({
        name: 'perfTestList',
        components: { IntroButton, vueHeadful, SearchBar, Vuetable, VuetablePagination },
    })
    export default class PerfTestList extends Mixins(Base, MessagesMixin, TableConfig) {
        runningSummary = `0 ${this.i18n('perfTest.list.runningSummary')}`;
        tests = [];
        autoUpdateTargets = [];
        queryFilter = new Set();
        sort = 'id,DESC';

        table = {
            css: {},
            appendParams: {},
            pagination: {
                perPage: 10,
            },
        };

        showTable = false;
        updateStatusTimeoutId = 0;

        created() {
            this.table.css = this.tableCss;
        }

        mounted() {
            this.init();
            this.$refs.vuetable.reload().then(() => this.showTable = true);
            this.updateStatusTimeoutId = setTimeout(this.updatePerftestStatus, 2000);
        }

        beforeDestroy() {
            clearTimeout(this.updateStatusTimeoutId);
        }

        init() {
            this.$refs.searchBar.searchText = this.$route.query.query || '';
            this.$refs.searchBar.selectedTag = this.$route.query.tag || '';
            if (this.$route.query.queryFilter) {
                const queryFilters = this.$route.query.queryFilter.split('');
                queryFilters.forEach(filterToken => {
                    if (filterToken === 'R') {
                        this.$refs.searchBar.running = true;
                    }
                    if (filterToken === 'S') {
                        this.$refs.searchBar.scheduled = true;
                    }
                    this.queryFilter.add(filterToken);
                });
            }
            this.$refs.vuetable.currentPage = parseInt(this.$route.query['page.page']) || 1;
            this.table.pagination.perPage = parseInt(this.$route.query['page.size']) || this.table.pagination.perPage;
            this.sort = this.$route.query.sort || this.sort;
        }

        changePage(nextPage) {
            this.$refs.vuetable.changePage(nextPage);
            history.replaceState('', '', this.makeQueryString(this.$refs.vuetable.currentPage, this.table.pagination.perPage,
                this.$refs.searchBar.searchText, this.makeQueryFilter(), this.sort, this.$refs.searchBar.selectedTag));
        }

        transform(data) {
            this.tests = data.tests;
            return data;
        }

        beforePagination(paginationData) {
            paginationData.total = paginationData.totalElements;
            paginationData.last_page = Math.ceil(paginationData.total / this.table.pagination.perPage);
            paginationData.per_page = this.table.pagination.perPage;
            paginationData.current_page = this.$refs.vuetable.currentPage;
            this.$refs.pagination.setPaginationData(paginationData);
        }

        beforeTableLoading() {
            this.table.appendParams['page.page'] = this.$refs.vuetable.currentPage - 1;
            this.table.appendParams['page.size'] = this.table.pagination.perPage;
            this.table.appendParams.queryFilter = this.makeQueryFilter();
            this.table.appendParams.query = this.$refs.searchBar.searchText;
            this.table.appendParams.tag = this.$refs.searchBar.selectedTag;
            if (this.$refs.vuetable.getSortParam()) {
                this.sort = this.$refs.vuetable.getSortParam().split('|').join(',');
            }
            this.table.appendParams.sort = this.sort;
        }

        tableLoaded() {
            this.autoUpdateTargets = [];
            this.tests.forEach((test, index) => {
                if (!this.isFinishedStatusType(test)) {
                    this.autoUpdateTargets.push({ 'id': test.id, 'index': index });
                }
            });
            this.$nextTick(() => {
                $('[data-toggle="popover"]').popover('dispose');
                $('[data-toggle="popover"]').popover();
            });
        }

        updateTableWithUrl() {
            history.replaceState('', '', this.makeQueryString(1, this.table.pagination.perPage, this.$refs.searchBar.searchText,
                this.makeQueryFilter(), this.sort, this.$refs.searchBar.selectedTag));
            this.$refs.vuetable.refresh();
        }

        makeQueryString(page, pageSize, query, queryFilter, sort, tag) {
            return `${this.contextPath}/perftest?page.page=${page}&page.size=${pageSize}&query=${query}&queryFilter=${queryFilter}&sort=${sort}&tag=${tag}`;
        }

        isFinishedStatusType(test) {
            return test.status.name === 'FINISHED' || test.status.name === 'STOP_BY_ERROR' ||
                test.status.name === 'STOP_ON_ERROR' || test.status.name === 'CANCELED';
        }

        getOwnerPopoverContent(test) {
            let content = `${this.i18n('perfTest.list.owner')} : ${test.createdUser.userName} (${test.createdUser.userId})`;
            if (test.lastModifiedUser) {
                content += `<br> ${this.i18n('perfTest.list.modifier.oneLine')} : ${test.lastModifiedUser.userName} (${test.lastModifiedUser.userId})`;
            }
            return content;
        }

        getErrorRatePopoverContent(test) {
            return `${this.i18n('perfTest.list.totalTests')} : ${test.tests + test.errors}<br>` +
                `${this.i18n('perfTest.list.successfulTests')} : ${test.tests}<br>` +
                `${this.i18n('perfTest.list.errors')} : ${test.errors}`;
        }

        getVuserPopoverContent(test) {
            return `${this.i18n('perfTest.list.agent')} : ${test.agentCount ? test.agentCount : 0}<br>` +
                `${this.i18n('perfTest.list.process')} : ${test.processes ? test.processes : 0}<br>` +
                `${this.i18n('perfTest.list.thread')} : ${test.threads ? test.threads : 0}`;
        }

        getTestNamePopoverContent(test) {
            let content = `${test.description} <p>${test.testComment}</p>`;
            if (test.scheduledTime) {
                content += `${this.i18n('perfTest.list.scheduledTime')} : ${this.$options.filters.dateFormat(test.scheduledTime, 'YYYY-MM-DD HH:mm')}<br>`;
            }
            content += `${this.i18n('perfTest.list.modifiedTime')} : ${this.$options.filters.dateFormat(test.lastModifiedDate, 'YYYY-MM-DD HH:mm')}<br>`;
            if (test.tagString) {
                content += `${this.i18n('perfTest.config.tags')} : ${test.tagString}<br>`;
            }
            content += this.getOwnerPopoverContent(test);
            return content;
        }

        getErrorRate(tests, errors) {
            if (!this.$utils.exists(tests) || !this.$utils.exists(errors)) {
                return '';
            }

            if (tests === 0 && errors === 0) {
                return '0.0%';
            }

            return `${(errors / (tests + errors) * 100).toFixed(1)}%`;
        }

        runQueryFilter(queryFilter) {
            queryFilter.enable ? this.queryFilter.add(queryFilter.token) : this.queryFilter.delete(queryFilter.token);
            history.replaceState('', '', this.makeQueryString(0, this.table.pagination.perPage, this.$refs.searchBar.searchText,
                this.makeQueryFilter(), this.sort, this.$refs.searchBar.selectedTag));
            this.$refs.vuetable.refresh();
        }

        makeQueryFilter() {
            let queryFilterString = '';
            this.queryFilter.forEach(val => queryFilterString += val);
            return queryFilterString;
        }

        deleteTests(ids) {
            this.$bootbox.confirm({
                message: this.i18n('perfTest.message.delete.confirm'),
                buttons: {
                    confirm: { label: this.i18n('common.button.ok') },
                    cancel: { label: this.i18n('common.button.cancel') },
                },
                onConfirm: () => {
                    if (ids) {
                        this.$http.delete('/perftest/api', {
                            params: {
                                ids,
                            },
                        }).then(res => {
                            if (res.data.success) {
                                this.showSuccessMsg(this.i18n('perfTest.message.delete.success'));
                                this.$refs.vuetable.refresh();
                            }
                        }).catch(() => this.showErrorMsg(this.i18n('perfTest.message.delete.error')));
                    }
                },
            });
        }

        stopTest(id) {
            this.$bootbox.confirm({
                message: this.i18n('perfTest.message.stop.confirm'),
                buttons: {
                    confirm: { label: this.i18n('common.button.ok') },
                    cancel: { label: this.i18n('common.button.cancel') },
                },
                onConfirm: () => this.$http.put(`/perftest/api/${id}?action=stop`)
                    .then(() => this.showSuccessMsg(this.i18n('perfTest.message.stop.success')))
                    .catch(() => this.showErrorMsg(this.i18n('perfTest.message.stop.error'))),
            });
        }

        showChart(index) {
            this.$refs.vuetable.toggleDetailRow(index);
        }

        updatePerftestStatus() {
            const ids = this.autoUpdateTargets.map(test => test.id).join(',');
            if (ids) {
                this.$http.get('/perftest/api/status', {
                    params: {
                        ids,
                    },
                }).then(res => {
                    const statuses = res.data.status.reverse();

                    this.autoUpdateTargets.forEach((target, index) => {
                        const updatedStatus = statuses[index].status;
                        const message = statuses[index].message;

                        if (updatedStatus.reportable) {
                            this.$refs.vuetable.refresh();
                        }

                        this.tests[target.index].status = updatedStatus;
                        this.runningSummary = `${res.data.perfTestInfo.length} ${this.i18n('perfTest.list.runningSummary')}`;

                        const $ball = $(`#ball_${target.id}`);
                        $ball.attr('title', updatedStatus.name);
                        $ball.attr('data-content', message);
                    });
                }).finally(() => this.updateStatusTimeoutId = setTimeout(this.updatePerftestStatus, 2000));
            } else {
                this.updateStatusTimeoutId = setTimeout(this.updatePerftestStatus, 2000);
            }
        }
    }
</script>

<style lang="less">
    .perftest-list-container {
        table {
            th {
                padding-left: 5px;
                padding-right: 5px;
            }
        }
    }
</style>

<style lang="less" scoped>
    .perftest-list-container {
        margin-bottom: 80px;

        .img-unit {
            height: 110px;
            padding: 0;
            margin-top: 0;
        }

        table {
            font-size: 12px;
            width: 940px;
            table-layout: fixed;
            margin-bottom: 6px;

            .ball {
                img {
                    width: 23px;
                    height: 23px;
                }
            }
        }

        .current-running-status-container {
            text-align: right;
            margin-top: -19px;

            code {
                padding: 0 4px;
                border-radius: 2px;
                background-color: white;
                width: 300px;
            }
        }

        .intro-button-container {
            margin-top: -26px;
        }

        .popover {
            width: auto;
            min-width: 200px;
            max-width: 600px;
            max-height: 500px;
        }
    }

    .intro-button-container {
        .intro-button-title {
            margin-right: -30px;
        }
    }
</style>
