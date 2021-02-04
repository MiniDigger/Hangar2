<template>
    <v-data-table
        :headers="headers"
        :items="authors.result"
        :options.sync="options"
        :server-items-length="authors.pagination.count"
        :loading="loading"
        :items-per-page="10"
        class="elevation-1"
    >
        <template #item.roles="{ item }">
            {{ item.roles.map((r) => r.title).join(', ') }}
        </template>
    </v-data-table>
</template>

<script lang="ts">
import { Component, Vue, Watch } from 'nuxt-property-decorator';
import { Context } from '@nuxt/types';
import { PaginatedResult, User } from 'hangar-api';
import { DataOptions } from 'vuetify';

@Component({
    head: {
        title: 'Authors',
    },
})
export default class AuthorsPage extends Vue {
    headers = [
        { text: 'Username', value: 'name' },
        { text: 'Roles', value: 'roles' },
        { text: 'Joined', value: 'joinDate' },
        { text: 'Projects', value: 'projectCount' },
    ];

    authors?: PaginatedResult<User>;
    loading = false;
    options = { page: 1, itemsPerPage: 10 } as DataOptions;
    initialLoad = true;

    @Watch('options', { deep: true })
    onOptionsChanged() {
        if (this.initialLoad) {
            this.initialLoad = false;
            return;
        }
        this.loading = true;

        this.$api.request<PaginatedResult<User>>('authors', false, 'get', this.requestOptions).then((authors) => {
            this.authors = authors;
            this.loading = false;
        });
    }

    get requestOptions() {
        if (!this.options) {
            return {};
        }

        let sort = '';
        if (this.options.sortBy.length === 1) {
            sort = this.options.sortBy[0];
            if (this.options.sortDesc[0]) {
                sort = '-' + sort;
            }
        }
        return {
            limit: this.options.itemsPerPage,
            offset: (this.options.page - 1) * this.options.itemsPerPage,
            sort,
        };
    }

    async asyncData({ $api }: Context): Promise<{ authors: PaginatedResult<User> }> {
        return { authors: await $api.request<PaginatedResult<User>>('authors', false, 'get', { limit: 10, offset: 0 }) };
    }
}
</script>

<style lang="scss" scoped></style>