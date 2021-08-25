<template>

    <div v-if="notification.display == true" class="notification" :class="notification.class">
        <p class="notification-text">{{ notification.message }}</p>
    </div>

    <div :class="name" class="table-container">

        <div class="table-actions">

            <div v-if="routes.crud?.create" class="table-action table-action-create">
                <a :href="this.$route(routes.crud.create.name)"><i v-if="routes.crud?.create?.icon" :class="routes.crud.create.icon"></i>Create</a>
            </div>

            <div v-if="searchable.length > 0 && show.includes('search')" class="table-action table-action-search">
                <input v-model="query.q" type="text" placeholder="Search..." />
            </div>
            
            <div v-if="show.includes('limit')" class="table-action table-action-limit">
                <select v-model="limit" class="custom-select">
                    <option v-for="value of resultsPerPageOptions" :key="value" :value="value">{{ value }}</option>
                </select>
            </div>

            <div v-if="routes.crud?.destroy?.bulk" class="table-action table-action-delete-bulk">
                <form @submit="destroy($event, selectedIds)">
                    <button :disabled="selectedIds.length <= 0" type="submit" :class="{ disabled: selectedIds.length <= 0 }">Delete selected</button>
                </form>
            </div>
            
        </div>
        
        <table class="table">

            <thead>
                <tr>
                    <th v-if="routes.crud?.destroy?.bulk"></th>

                    <th v-for="(header, headerIndex) in headers" :key="headerIndex">
                        <a v-if="header.orderable" class="orderable" :class="{ 'direction-asc': (query.direction == 'asc'), 'direction-desc': (query.direction == 'desc')}" href="#" @click="toggleOrder($event, header.column)">{{ header.display }}</a>
                        <template v-else>{{ header.display }}</template>
                    </th>

                    <th v-if="routes.crud?.edit || routes.crud?.show || routes.crud?.destroy">Actions</th>
                </tr>
            </thead>

            <tbody :class="{ loading: loading }">

                <tr v-if="tableData.length <= 0">
                    <td v-if="routes.crud?.destroy?.bulk"></td>

                    <td v-for="(header, headerIndex) in headers" :key="headerIndex">
                        <template v-if="headerIndex == 0">No results</template>
                    </td>

                    <td v-if="routes.crud"></td>
                </tr>
                
                <tr v-else v-for="(row, rowIndex) in tableData" :key="rowIndex">

                    <td v-if="routes.crud?.destroy?.bulk" class="row-action row-action-select">
                        <input type="checkbox" :value="row.id" v-model="selectedIds" />
                    </td>

                    <template v-for="(columnData, column) in row" :key="column">
                        <td>{{ columnData }}</td>
                    </template>

                    <td v-if="routes.crud?.edit || routes.crud?.show || routes.crud?.destroy" class="row-actions">
                        <a v-if="routes.crud?.show" :href="this.$route(routes.crud.show.name, row.id)" class="row-action row-action-show"><i :class="routes.crud?.show?.icon ?? 'fas fa-fw fa-eye'"></i></a>
                        <a v-if="routes.crud?.edit" :href="this.$route(routes.crud.edit.name, row.id)" class="row-action row-action-edit"><i :class="routes.crud?.edit?.icon ?? 'fas fa-fw fa-edit'"></i></a>
                        <form v-if="routes.crud?.destroy" @submit="destroy($event, row.id)" class="row-action row-action-delete">
                            <button type="submit"><i :class="routes.crud?.destroy?.icon ?? 'fas fa-fw fa-trash'"></i></button>
                        </form>
                    </td>
                </tr>

            </tbody>

        </table>

        <div v-if="tableData.length > 0 && show.includes('pagination')" class="pagination-container">
            <p class="pagination-status">Results: {{ paginationStatus.from }} - {{ paginationStatus.to }} of {{ paginationStatus.total }}</p>
            <ul class="pagination">
                <template v-for="(link, linkIndex) in paginationData" :key="linkIndex">
                    <li class="pagepagination-item" :class="{ disabled: link.url == null, active: link.active }">
                        <a class="pageination-link" :data-href="link.url" v-html="link.label" @click="link.url != null && paginate($event)"></a>
                    </li>
                </template>
            </ul>
        </div>

    </div>

</template>

<script>
import _ from 'lodash';

export default {

    props: {

        /**
         * Required table identifier
         * .table-container will also have the class .table-name 
         */ 
        name: {
            type: String,
            required: true
        },

        /* 
        * Routes (data route + CRUD routes)
        * Example: :route="{
        *   data: 'api.usres', 
        *   crud: { 
        *       create: 'data.create', 
        *       show: 'data.show', 
        *       icon: 'fas fa-user' 
        *   }
        * }"
        * 
        * The above example will show a create button in the top, and a view button
        * for each table row. If you have the routes edit, show and destroy included,
        * remember they need to have icon classes! 
        * Icon is optional for create route.
        * Destroy route can also contain a boolean "bulk" field to enable checkboxes and
        * buld deletion of rows
        */
        routes: {
            type: Object,
            required: true
        },

        // All headers must be present with a corresponding
        // database column
        // Example: [{ display: 'Username', column: 'name'}]
        // Example: [{ display: 'Became a member', column: 'created_at'}]
        headers: {
            type: Array,
            required: true
        },

        // How many rows to display at a time
        // Defaults to 25 is none is passed
        resultsPerPage: {
            type: Number,
            required: false,
            default: 25
        },

        // Select box for how many rows per page we want
        // Has default value, not required
        resultsPerPageOptions: {
            type: Array,
            required: false,
            default: [5, 10, 25, 50, 100, 250, 500, 1000]
        },

        // Wich "modules" to show
        // Only 3 accepted values for now.
        // search, pagination and limit
        show: {
            type: Array,
            required: false,
            default: function() {
                return ['search', 'pagination', 'limit']
            }
        },

        // How long to show notifications in ms,
        // defaults to 500ms
        notificationTimeout: {
            type: Number,
            required: false,
            default: function() {
                return 500
            }
        }

    },

    data: function() {
        return {

            // All table data
            tableData: {},

            // Generated table headers
            tableHeaders: [],

            // Pagination data
            paginationData: {},

            // Pagination status
            paginationStatus: {},

            // Loading state
            loading: false,

            // Current page
            current: 1,

            // Query 
            query: {
                direction: false,
                order: false,
                q: ''
            },

            // Searchable column
            // The data for this array
            // is generated from the headers
            // in the mounted() method below
            searchable: [],

            // Limit model
            limit: this.resultsPerPage,

            // Selected model
            selectedIds: [],

            // Notification (shown on deletion)
            notification: {
                display: false,
                message: false,
                class: ''
            }

        }
    },

    beforeMount() {
                
        // Create searchable columns array out of
        // the headers given
        for (const [headerIndex, header] of Object.entries(this.headers)) {
            (header.searchable == true) && this.searchable.push(header.column); 
        }

        // Once mounted get the first results
        this.getResults();

    },

    watch: {

        // We do not want to fire a search every time we a character,
        // instead we debounce it by 300ms after we stop typing
        'query.q': _.debounce(function(query) {
            this.current = 1;
            this.getResults();
        }, 300),

        // Update how many rows to show
        limit: function() {
            this.current = 1;
            this.getResults();
        }

    },

    methods: {

        // Toggle order directon/order
        toggleOrder(event, key) {
            event.preventDefault();

            if (!this.query.direction || this.query.order != key) {
                this.query.direction = 'asc';
                this.query.order = key;
            }

            else if (this.query.direction == 'asc') {
                this.query.direction = 'desc';
                this.query.order = key;
            }

            else if (this.query.direction == 'desc') {
                this.query.direction = false;
                this.query.order = false;
            }

            // Query direction/order is updated
            // Refresh the table
            this.getResults();
        },

        // Simple pagination based on what link got clicked in the pagination
        // elements. 
        // Search query are also beeing added to the URL if there
        // has been a search
        paginate(event) {
            event.preventDefault();
            let url = new URL(event.target.getAttribute('data-href'));
            this.current = url.searchParams.get('page');
            this.getResults();
        },

        // This is our main function for grabbing
        // data from the backend
        getResults() {

            this.loading = true;

            // Prepare the URL
            let url = new URL(this.$route(this.routes.data));
            url.searchParams.set('page', this.current);
            url.searchParams.set('limit', this.limit);

            // Add order parameters if they exist
            if (this.query.order || this.query.direction) {
                url.searchParams.set('sortBy', this.query.order);
                url.searchParams.set('sortDirection', this.query.direction);
            }

            // Add search parameters if there exists a search
            if (this.query.q.length) {
                url.searchParams.set('q', this.query.q);
                url.searchParams.set('where', this.searchable);
            }

            // Do the xhr request
            this.$axios.get(url)
            .then(response => {
                this.tableData = response.data.data;
                this.paginationData = response.data.links;
                this.paginationStatus = { 
                    to: response.data.to,
                    from: response.data.from,
                    total: response.data.total
                };
                this.loading = false;
            })
            .catch((error) => {
                throw new Error(error);
            });

        },
        
        // Simple delete method
        destroy(event, model) {
            event.preventDefault()

            if (this.routes.crud.destroy) {

                // Confirmation text to be sure that the user
                // wants to delete the model
                let confirmText = (Array.isArray(model))
                ? 'Are you sure you want to delete the selected models'
                : 'Are you sure you want to delete this model?'

                // Check if the model is an array of models or a simple
                // model (integer)
                model = (Array.isArray(model))
                ? model.map((v, k) => { return v }).join()
                : model

                // Confirm action
                if (confirm(confirmText)) {
                    this.$axios.post(this.$route(this.routes.crud.destroy.name, model), {
                        _method: 'DELETE'
                    })
                    .then((response) => {

                        // Reset selected rows to an empty array
                        this.selectedIds = []

                        var that = this;
                        this.notification.message = response.data.message;
                        this.notification.type = response.data.type;
                        this.notification.display = true;
                        setTimeout(() => {
                            that.notification.message,
                            that.notification.type,
                            that.notification.display = false;
                        }, that.notificationTimeout)

                        // Refresh the table
                        this.getResults()

                    })
                    .catch((error) => {
                        throw new Error(error);
                    })
                }
            }
        },

    },

}
</script>