<template>
    <div class="py-2 line font-monospace" :class="{['log-border-' + log.level.toLowerCase()]: cursor && log.level !== undefined}" v-if="filtered">
        <el-icon v-if="cursor" class="icon_container" :style="{color: iconColor}" :size="25">
            <MenuRight />
        </el-icon>
        <span :class="levelClasses" class="border header-badge log-level el-tag noselect">{{ log.level }}</span>
        <div class="log-content d-inline-block">
            <span v-if="title" class="fw-bold">{{ (log.taskId ?? log.flowId ?? "") }}</span>
            <div
                class="header"
                :class="{'d-inline-block': metaWithValue.length === 0, 'me-3': metaWithValue.length === 0}"
            >
                <span class="header-badge text-secondary">
                    {{ $filters.date(log.timestamp, "iso") }}
                </span>
                <span v-for="(meta, x) in metaWithValue" :key="x">
                    <span class="header-badge property">
                        <span>{{ meta.key }}</span>
                        <template v-if="meta.router">
                            <router-link :to="meta.router">{{ meta.value }}</router-link>
                        </template>
                        <template v-else>
                            {{ meta.value }}
                        </template>
                    </span>
                </span>
            </div>
            <v-runtime-template :template="markdownRenderer" />
        </div>
    </div>
</template>
<script>
    import Convert from "ansi-to-html"
    import xss from "xss";
    import Markdown from "../../utils/markdown";
    import VRuntimeTemplate from "vue3-runtime-template";
    import MenuRight from "vue-material-design-icons/MenuRight.vue";


    let convert = new Convert();

    export default {
        components:{
            VRuntimeTemplate,
            MenuRight
        },
        props: {
            cursor: {
                type: Boolean,
                default: false
            },
            log: {
                type: Object,
                required: true,
            },
            filter: {
                type: String,
                default: "",
            },
            level: {
                type: String,
                required: true,
            },
            excludeMetas: {
                type: Array,
                default: () => [],
            },
            title: {
                type: Boolean,
                default: false
            }
        },
        data() {
            return {
                markdownRenderer: undefined
            }
        },
        async created() {
            this.markdownRenderer = await this.renderMarkdown();
        },
        computed: {
            metaWithValue() {
                const metaWithValue = [];
                const excludes = [
                    "message",
                    "timestamp",
                    "thread",
                    "taskRunId",
                    "level",
                    "index",
                    "attemptNumber"
                ];
                excludes.push.apply(excludes, this.excludeMetas);
                for (const key in this.log) {
                    if (this.log[key] && !excludes.includes(key)) {
                        let meta = {key, value: this.log[key]};
                        if (key === "executionId") {
                            meta["router"] = {
                                name: "executions/update", params: {
                                    namespace: this.log["namespace"],
                                    flowId: this.log["flowId"],
                                    id: this.log[key]
                                }
                            };
                        }

                        if (key === "namespace") {
                            meta["router"] = {name: "flows/list", query: {namespace: this.log[key]}};
                        }


                        if (key === "flowId") {
                            meta["router"] = {
                                name: "flows/update",
                                params: {namespace: this.log["namespace"], id: this.log[key]}
                            };
                        }

                        metaWithValue.push(meta);
                    }
                }
                return metaWithValue;
            },
            levelClasses() {
                const lowerCaseLevel = this.log?.level?.toLowerCase();
                return `log-content-${lowerCaseLevel} log-border-${lowerCaseLevel} log-bg-${lowerCaseLevel}`;
            },
            filtered() {
                return (
                    this.filter === "" || (
                        this.log.message &&
                        this.log.message.toLowerCase().includes(this.filter)
                    )
                );
            },
            iconColor() {
                const logLevel = this.log.level?.toLowerCase();
                return `var(--log-content-${logLevel}) !important`; // Use CSS variable for icon color
            },
            message() {
                let logMessage = !this.log.message ? "" : convert.toHtml(xss(this.log.message, {
                    allowList: {"span": ["style"]}
                }));

                logMessage = logMessage.replaceAll(
                    /(['"]?)(https?:\/\/[^'"\s]+)(['"]?)/g,
                    "$1<a href='$2' target='_blank'>$2</a>$3"
                );
                return logMessage;
            }
        },
        methods: {
            async renderMarkdown() {
                let markdown = await Markdown.render(this.message, {onlyLink: true});

                // Avoid rendering non-existent properties in the template by VRuntimeTemplate
                markdown = markdown.replace(/{{/g, "&#123;&#123;").replace(/}}/g, "&#125;&#125;");

                return markdown
            },
        },
    };
</script>
<style scoped lang="scss">
    @import "@kestra-io/ui-libs/src/scss/variables";

    div.line {
        cursor: text;
        white-space: pre-wrap;
        word-break: break-all;
        display: flex;
        align-items: center;
        gap: $spacer;

        border-left-width: 2px !important;
        border-left-style: solid;
        border-left-color: transparent;

        .icon_container{
            margin-left: -0.90rem;
        }
        
        .log-level {
            padding: calc(var(--spacer) / 4);
        }

        .log-content {
            .header > * + * {
                margin-left: $spacer;
            }
        }

        .el-tag {
            height: auto;
        }

        .header-badge {
            font-size: 95%;
            text-align: center;
            white-space: nowrap;
            vertical-align: baseline;
            width: 40px;

            span:first-child {
                margin-right: 6px;
                font-family: var(--bs-font-sans-serif);
                user-select: none;

                &::after {
                    content: ":";
                }
            }

            & a {
                border-radius: var(--bs-border-radius);
            }

            &.log-level {
                white-space: pre;
                border-radius: 4px;
            }
        }

        .noselect {
            user-select: none;
            color: $white;

            html:not(.dark) & {
                color: $black;
            }
        }

        .message {
            line-height: 1.8;
        }

        p, :deep(.log-content p) {
            display: inline;
            margin-bottom: 0;
        }
    }
</style>
