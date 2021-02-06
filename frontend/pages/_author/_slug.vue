<template>
    <div>
        <template v-if="!isPublic">
            <!-- todo alert for visibility stuff -->
            <v-alert v-if="needsChanges" type="error">
                <!-- todo remove is no EditPage perm -->
                <v-btn type="primary" :to="'/' + slug + '/manage/sendforapproval'">{{ $t('project.sendForApproval') }} </v-btn>
                <strong>{{ $t('visibility.notice.' + project.visibility) }}</strong>
                <br />
                <Markdown :input="visibilityComment ? visibilityComment : 'Unknown'" />
            </v-alert>
            <v-alert v-else-if="isSoftDeleted" type="error">
                <!-- todo insert lastVisibilityChangeUser here -->
                {{ $t('visibility.notice.' + project.visibility, ['Admin']) }}
            </v-alert>
            <v-alert v-else type="error">
                {{ $t('visibility.notice.' + project.visibility) }}
                <Markdown v-if="visibilityComment" :input="visibilityComment" />
            </v-alert>
        </template>
        <v-row>
            <v-col cols="1">
                <UserAvatar :username="project.namespace.owner" :avatar-url="$util.avatarUrl(project.namespace.owner)" clazz="user-avatar-sm"></UserAvatar>
            </v-col>
            <v-col cols="auto">
                <h1 class="d-inline">
                    <NuxtLink :to="'/' + project.namespace.owner">{{ project.namespace.owner }}</NuxtLink> /
                    <NuxtLink :to="'/' + slug">{{ project.name }}</NuxtLink>
                </h1>
                <div>
                    <v-subheader>{{ project.description }}</v-subheader>
                </div>
            </v-col>
            <v-spacer />
            <v-col cols="3">
                <!-- todo if author, disable -->
                <v-btn @click="toggleStar">
                    <v-icon v-if="project.userActions.starred">mdi-star</v-icon>
                    <v-icon v-else>mdi-star-outline</v-icon>
                </v-btn>
                <!-- todo if author, remove -->
                <v-btn @click="toggleWatch">
                    <template v-if="project.userActions.watching">
                        <v-icon>mdi-eye-off</v-icon>
                        {{ $t('project.actions.unwatch') }}
                    </template>
                    <template v-else>
                        <v-icon>mdi-eye</v-icon>
                        {{ $t('project.actions.watch') }}
                    </template>
                </v-btn>
                <!-- todo if not logged in or author, remove both -->
                <FlagModal ref="flagModal" :project="project" />
                <v-btn @click="showFlagModal">
                    <v-icon>mdi-flag</v-icon>
                    {{ $t('project.actions.flag') }}
                </v-btn>
                <!-- todo only enable if has perms -->
                <v-menu bottom offset-y>
                    <template #activator="{ on, attrs }">
                        <v-btn v-bind="attrs" v-on="on">
                            {{ $t('project.actions.adminActions') }}
                        </v-btn>
                    </template>
                    <v-list-item :to="'/' + slug + '/flags'">
                        <v-list-item-title>
                            {{ $t('project.actions.flagHistory', []) }}
                        </v-list-item-title>
                    </v-list-item>
                    <v-list-item :to="'/' + slug + '/notes'">
                        <v-list-item-title>
                            {{ $t('project.actions.staffNotes', []) }}
                        </v-list-item-title>
                    </v-list-item>
                    <v-list-item :to="'/admin/log/?projectFilter=' + slug">
                        <v-list-item-title>
                            {{ $t('project.actions.userActionLogs') }}
                        </v-list-item-title>
                    </v-list-item>
                    <v-list-item :href="$util.forumUrl(project.namespace.owner)">
                        <v-list-item-title>
                            {{ $t('project.actions.forum') }}
                            <v-icon>mdi-open-in-new</v-icon>
                        </v-list-item-title>
                    </v-list-item>
                </v-menu>
            </v-col>
        </v-row>
        <v-row>
            <v-tabs active-class="other-class">
                <v-tab
                    v-for="tab in tabs"
                    :key="tab.title"
                    exact
                    :to="tab.external ? null : tab.link"
                    :href="tab.external ? tab.link : null"
                    :nuxt="!tab.external"
                >
                    <v-icon>{{ tab.icon }}</v-icon>
                    {{ tab.title }}
                    <v-icon v-if="tab.external">mdi-open-in-new</v-icon>
                </v-tab>
            </v-tabs>
        </v-row>
        <NuxtChild class="mt-4">
            <v-tab-item>
                {{ $route.name }}
            </v-tab-item>
        </NuxtChild>
    </div>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator';
import { Context } from '@nuxt/types';
import { HangarApiException, Project } from 'hangar-api';
import { AxiosError } from 'axios';
import Markdown from '~/components/Markdown.vue';
import FlagModal from '~/components/FlagModal.vue';
import UserAvatar from '~/components/UserAvatar.vue';
import { Visibility } from '~/types/enums';

interface Tab {
    title: String;
    icon: String;
    link: String;
    external: Boolean;
}

@Component({
    components: {
        FlagModal,
        Markdown,
        UserAvatar,
    },
})
export default class ProjectPage extends Vue {
    project!: Project;

    head() {
        return {
            title: this.project?.name,
        };
    }

    async asyncData({ $api, params, error }: Context): Promise<{ project: Project } | void> {
        return await $api
            .request<Project>(`projects/${params.author}/${params.slug}`)
            .then((project) => {
                return Promise.resolve({ project });
            })
            .catch((err: AxiosError) => {
                if (err.response?.data?.isHangarApiException) {
                    const hangarError: HangarApiException = err.response?.data as HangarApiException;
                    error({
                        message: hangarError.message,
                        statusCode: hangarError.httpError.statusCode,
                    });
                }
            });
    }

    get tabs(): Tab[] {
        const tabs = [] as Tab[];
        tabs.push({ title: this.$t('project.tabs.docs') as String, icon: 'mdi-book', link: this.slug, external: false });
        tabs.push({ title: this.$t('project.tabs.versions') as String, icon: 'mdi-download', link: this.slug + '/versions', external: false });
        // todo check if has a discussion
        tabs.push({ title: this.$t('project.tabs.discuss') as String, icon: 'mdi-account-group', link: this.slug + '/discuss', external: false });
        // todo check if settings perm
        tabs.push({ title: this.$t('project.tabs.settings') as String, icon: 'mdi-cog', link: this.slug + '/settings', external: false });
        if (this.project.settings.homepage) {
            tabs.push({ title: this.$t('project.tabs.homepage') as String, icon: 'mdi-home', link: this.project.settings.homepage, external: true });
        }
        if (this.project.settings.issues) {
            tabs.push({ title: this.$t('project.tabs.issues') as String, icon: 'mdi-bug', link: this.project.settings.issues, external: true });
        }
        if (this.project.settings.sources) {
            tabs.push({ title: this.$t('project.tabs.source') as String, icon: 'mdi-code-tags', link: this.project.settings.sources, external: true });
        }
        if (this.project.settings.support) {
            tabs.push({ title: this.$t('project.tabs.support') as String, icon: 'mdi-chat-question', link: this.project.settings.support, external: true });
        }
        return tabs;
    }

    get isPublic(): Boolean {
        return this.project.visibility === Visibility.PUBLIC;
    }

    get needsChanges(): Boolean {
        return this.project.visibility === Visibility.NEEDS_CHANGES;
    }

    get isSoftDeleted(): Boolean {
        return this.project.visibility === Visibility.SOFT_DELETE;
    }

    get visibilityComment() {
        // todo get last visibility change comment
        return null;
    }

    get slug(): String {
        return `/${this.project.namespace.owner}/${this.project.namespace.slug}`;
    }

    toggleStar() {
        // TODO toggle star
    }

    toggleWatch() {
        // todo toggle watch
    }

    $refs!: {
        flagModal: FlagModal;
    };

    showFlagModal() {
        this.$refs.flagModal.show();
    }
}
</script>