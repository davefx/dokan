<template>
    <div>
        <UpgradeBanner v-if="! hasPro"></UpgradeBanner>

        <div class="withdraw-requests">
            <h1>{{ __( 'Withdraw Requests', 'dokan-lite' ) }}</h1>

            <modal
                :title="__( 'Update Note', 'dokan-lite' )"
                v-if="showModal"
                @close="showModal = false"
            >
                <template slot="body">
                    <textarea v-model="editing.note" rows="3"></textarea>
                </template>

                <template slot="footer">
                    <button class="button button-primary button-large" @click="updateNote()">{{ __( 'Update Note', 'dokan-lite' ) }}</button>
                </template>
            </modal>

            <ul class="subsubsub">
                <li><router-link :class="{current: currentStatus === 'pending'}" :to="{ name: 'Withdraw', query: { status: 'pending' }}" >{{ __( 'Pending', 'dokan-lite' ) }} <span class="count">{{ counts.pending }}</span></router-link> | </li>
                <li><router-link :class="{current: currentStatus === 'approved'}" :to="{ name: 'Withdraw', query: { status: 'approved' }}">{{ __( 'Approved', 'dokan-lite' ) }} <span class="count">{{ counts.approved }}</span></router-link> | </li>
                <li><router-link :class="{current: currentStatus === 'cancelled'}" :to="{ name: 'Withdraw', query: { status: 'cancelled' }}" >{{ __( 'Cancelled', 'dokan-lite' ) }} <span class="count">{{ counts.cancelled }}</span></router-link> | </li>
            </ul>

            <list-table
                :columns="columns"
                :rows="requests"
                :loading="loading"
                :action-column="actionColumn"
                :actions="actions"
                :show-cb="showCb"
                :bulk-actions="bulkActions"
                :not-found="notFound"
                :total-pages="totalPages"
                :total-items="totalItems"
                :per-page="perPage"
                :current-page="currentPage"
                :text="$root.listTableTexts()"
                @pagination="goToPage"
                @action:click="onActionClick"
                @bulk:click="onBulkAction"
            >
                <template slot="seller" slot-scope="data">
                    <img :src="data.row.user.gravatar" :alt="data.row.user.store_name" width="50">
                    <strong><a :href="vendorUrl(data.row.user.id)">{{ data.row.user.store_name ? data.row.user.store_name : __( '(no name)', 'dokan-lite' ) }}</a></strong>
                </template>

                <template slot="vendor" slot-scope="{ row }">
                    <router-link :to="'/vendors/' + row.vendor.id">
                        {{ row.vendor.name ? row.vendor.name : __('(no name)', 'dokan-lite') }}
                    </router-link>
                </template>

                <template slot="amount" slot-scope="data">
                    <currency :amount="data.row.amount"></currency>
                </template>

                <template slot="status" slot-scope="data">
                    <span :class="data.row.status">{{ data.row.status | capitalize }}</span>
                </template>

                <template slot="created" slot-scope="data">
                    {{ moment(data.row.created).format('MMM D, YYYY') }}
                </template>

                <template slot="method_details" slot-scope="data">
                    <div class="method_details_inner" v-html="getPaymentDetails(data.row.method, data.row.details)"></div>
                </template>

                <template slot="filters">
                    <select
                        id="filter-vendors"
                        style="width: 190px;"
                        :data-placeholder="__('Filter by vendor', 'dokan-lite')"
                    />
                    <button
                        v-if="filter.user_id"
                        type="button"
                        class="button"
                        @click="filter.user_id = 0"
                    >&times;</button>
                </template>

                <template slot="actions" slot-scope="data">
                    <template v-if="data.row.status === 'pending'">
                        <div class="button-group">
                            <button class="button button-small" @click.prevent="changeStatus('approved', data.row.id)" :title="__( 'Approve Request', 'dokan-lite' )"><span class="dashicons dashicons-yes"></span></button>
                            <button class="button button-small" @click.prevent="openNoteModal(data.row.note, data.row.id)" :title="__( 'Add Note', 'dokan-lite' )"><span class="dashicons dashicons-testimonial"></span></button>
                        </div>
                    </template>
                    <template v-else-if="data.row.status === 'approved'">
                        <div class="button-group">
                            <button class="button button-small" @click.prevent="openNoteModal(data.row.note, data.row.id)" :title="__( 'Add Note', 'dokan-lite' )"><span class="dashicons dashicons-testimonial"></span></button>
                        </div>
                    </template>
                    <template v-else>
                        <div class="button-group">
                            <button class="button button-small" @click.prevent="changeStatus('pending', data.row.id)" :title="__( 'Mark as Pending', 'dokan-lite' )"><span class="dashicons dashicons-backup"></span></button>
                            <button class="button button-small" @click.prevent="openNoteModal(data.row.note, data.row.id)" :title="__( 'Add Note', 'dokan-lite' )"><span class="dashicons dashicons-testimonial"></span></button>
                        </div>
                    </template>
                </template>
            </list-table>
        </div>
    </div>
</template>

<script>
let ListTable = dokan_get_lib('ListTable');
let Modal     = dokan_get_lib('Modal');
let Currency  = dokan_get_lib('Currency');

import $ from 'jquery';
import UpgradeBanner from "admin/components/UpgradeBanner.vue";


export default {

    name: 'Withdraw',

    components: {
        ListTable,
        Modal,
        Currency,
        UpgradeBanner,
    },

    data () {
        return {
            showModal: false,
            editing: {
                id: null,
                note: null
            },

            totalPages: 1,
            perPage: 10,
            totalItems: 0,
            filter: {
                user_id: 0
            },
            counts: {
                pending: 0,
                approved: 0,
                cancelled: 0
            },
            notFound: this.__( 'No requests found.', 'dokan-lite' ),
            massPayment: this.__( 'Paypal Mass Payment File is Generated.', 'dokan-lite' ),
            showCb: true,
            loading: false,
            columns: {
                'seller': { label: this.__( 'Vendor', 'dokan-lite' ) },
                'amount': { label: this.__( 'Amount', 'dokan-lite' ) },
                'status': { label: this.__( 'Status', 'dokan-lite' ) },
                'method_title': { label: this.__( 'Method', 'dokan-lite' ) },
                'method_details': { label: this.__( 'Details', 'dokan-lite' ) },
                'note': { label: this.__( 'Note', 'dokan-lite' ) },
                'created': { label: this.__( 'Date', 'dokan-lite' ) },
                'actions': { label: this.__( 'Actions', 'dokan-lite' ) },
            },
            requests: [],
            actionColumn: 'seller',
            hasPro: dokan.hasPro ? true : false
        };
    },

    watch: {
        '$route.query.status'() {
            this.filter.user_id = 0;
            this.clearSelection('#filter-vendors');
            this.fetchRequests();
        },

        '$route.query.page'() {
            this.fetchRequests();
        },

        '$route.query.user_id'() {
            this.fetchRequests();
        },

        'filter.user_id'(user_id) {
            if (user_id === 0) {
                this.clearSelection('#filter-vendors');
            }

            this.goTo(this.query);
        }
    },

    computed: {

        currentStatus() {
            return this.$route.query.status || 'pending';
        },

        currentPage() {
            let page = this.$route.query.page || 1;

            return parseInt( page );
        },

        actions() {
            if ( 'pending' == this.currentStatus ) {
                return [
                    {
                        key: 'trash',
                        label: this.__( 'Delete', 'dokan-lite' )
                    },
                    {
                        key: 'cancel',
                        label: this.__( 'Cancel', 'dokan-lite' )
                    }
                ];
            } else if ( 'cancelled' == this.currentStatus ) {
                return [
                    {
                        key: 'trash',
                        label: this.__( 'Delete', 'dokan-lite' )
                    },
                    {
                        key: 'pending',
                        label: this.__( 'Pending', 'dokan-lite' )
                    }
                ];
            } else {
                return [];
            }
        },

        bulkActions() {
            if ( 'pending' == this.currentStatus ) {
                return [
                    {
                        key: 'approved',
                        label: this.__( 'Approve', 'dokan-lite' )
                    },
                    {
                        key: 'cancelled',
                        label: this.__( 'Cancel', 'dokan-lite' )
                    },
                    {
                        key: 'delete',
                        label: this.__( 'Delete', 'dokan-lite' )
                    },
                    {
                        key: 'paypal',
                        label: this.__( 'Download PayPal mass payment file', 'dokan-lite' )
                    },
                ];
            } else if ( 'cancelled' == this.currentStatus ) {
                return [
                    {
                        key: 'pending',
                        label: this.__( 'Pending', 'dokan-lite' )
                    },
                    {
                        key: 'delete',
                        label: this.__( 'Delete', 'dokan-lite' )
                    },
                    {
                        key: 'paypal',
                        label: this.__( 'Download PayPal mass payment file', 'dokan-lite' )
                    },
                ];
            } else {
                return [
                    {
                        key: 'paypal',
                        label: this.__( 'Download PayPal mass payment file', 'dokan-lite' )
                    },
                ];
            }
        }
    },

    created() {
        this.fetchRequests();
    },

    mounted() {
        const self = this;

        $('#filter-vendors').selectWoo({
            ajax: {
                url: "".concat(dokan.rest.root, "dokan/v1/stores"),
                dataType: 'json',
                headers: {
                    "X-WP-Nonce" : dokan.rest.nonce
                },
                data(params) {
                    return {
                        search: params.term
                    };
                },
                processResults(data) {
                    return {
                        results: data.map((store) => {
                            return {
                                id: store.id,
                                text: store.store_name
                            };
                        })
                    };
                }
            }
        });

        $('#filter-vendors').on('select2:select', (e) => {
            self.filter.user_id = e.params.data.id;
        });
    },

    methods: {

        updatedCounts(xhr) {
            this.counts.pending   = parseInt( xhr.getResponseHeader('X-Status-Pending') );
            this.counts.approved  = parseInt( xhr.getResponseHeader('X-Status-Completed') );
            this.counts.cancelled = parseInt( xhr.getResponseHeader('X-Status-Cancelled') );
        },

        updatePagination(xhr) {
            this.totalPages = parseInt( xhr.getResponseHeader('X-WP-TotalPages') );
            this.totalItems = parseInt( xhr.getResponseHeader('X-WP-Total') );
        },

        vendorUrl(id) {
            if ( window.dokan.hasPro === '1' ) {
                return dokan.urls.adminRoot + 'admin.php?page=dokan#/vendors/' + id;
            }

            return dokan.urls.adminRoot + 'user-edit.php?user_id=' + id;
        },

        fetchRequests() {
            this.loading = true;
            let url = '/withdraw?per_page=' + this.perPage + '&page=' + this.currentPage + '&status=' + this.currentStatus;
            if (parseInt(this.filter.user_id) > 0) {
                url += '&user_id=' + this.filter.user_id;
            }

            dokan.api.get(url)
            .done((response, status, xhr) => {
                this.requests = response;
                this.loading = false;

                this.updatedCounts(xhr);
                this.updatePagination(xhr);
            });
        },

        goToPage(page) {
            this.$router.push({
                name: 'Withdraw',
                query: {
                    status: this.currentStatus,
                    page: page,
                    user_id: this.filter.user_id
                }
            });
        },

        goTo(page) {
            this.$router.push({
                name: 'Withdraw',
                query: {
                    status: this.currentStatus,
                    user_id: this.filter.user_id
                }
            });
        },

        updateItem(id, value) {
            let index = this.requests.findIndex(x => x.id == id);

            this.$set(this.requests, index, value);
        },

        changeStatus(status, id) {
            this.loading = true;

            dokan.api.put('/withdraw/' + id, { status: status })
            .done(response => {
                // this.requests = response;
                this.loading = false;
                // this.updateItem(id, response);
                this.fetchRequests();
            });
        },

        onActionClick(action, row) {
            if ( 'cancel' === action ) {
                this.changeStatus('cancelled', row.id);
            }

            if ( 'pending' === action ) {
                this.changeStatus('pending', row.id);
            }

            if ( 'trash' === action ) {
                if ( confirm( this.__( 'Are you sure?', 'dokan-lite' ) ) ) {
                    this.loading = true;

                    dokan.api.delete('/withdraw/' + row.id)
                    .done(response => {
                        this.loading = false;
                        this.fetchRequests();
                    });
                }
            }
        },

        getPaymentDetails(method, data) {
            let details = '—';

            if ( data[method] !== undefined ) {
                if ( 'paypal' === method || 'skrill' === method ) {
                    details = data[method].email || '';

                } else if ( 'bank' === method ) {
                    if ( data.bank.hasOwnProperty('ac_name') ) {
                        details = '<p>' + this.sprintf( this.__( 'Account Name: %s', 'dokan-lite' ), data.bank.ac_name ) + '</p>';
                    }

                    if ( data.bank.hasOwnProperty('ac_number') ) {
                        details += '<p>' + this.sprintf( this.__( 'Account Number: %s', 'dokan-lite' ), data.bank.ac_number ) + '</p>';
                    }

                    if ( data.bank.hasOwnProperty('bank_name') ) {
                        details += '<p>' + this.sprintf( this.__( 'Bank Name: %s', 'dokan-lite' ), data.bank.bank_name ) + '</p>';
                    }

                    if ( data.bank.hasOwnProperty('iban') ) {
                        details += '<p>' + this.sprintf( this.__( 'IBAN: %s', 'dokan-lite' ), data.bank.iban ) + '</p>';
                    }

                    if ( data.bank.hasOwnProperty('routing_number') ) {
                        details += '<p>' + this.sprintf( this.__( 'Routing Number: %s', 'dokan-lite' ), data.bank.routing_number ) + '</p>';
                    }

                    if ( data.bank.hasOwnProperty('swift') ) {
                        details += '<p>' + this.sprintf( this.__( 'Swift Code: %s', 'dokan-lite' ), data.bank.swift ) + '</p>';
                    }
                }
            }

            return dokan.hooks.applyFilters( 'dokan_get_payment_details', details, method, data );
        },

        moment(date) {
            return moment(date);
        },

        onBulkAction(action, items) {
            let self = this;

            if ( _.contains(['delete', 'approved', 'cancelled', 'pending'], action) ) {

                let jsonData = {};
                jsonData[action] = items;

                this.loading = true;

                dokan.api.put('/withdraw/batch', jsonData)
                .done(response => {
                    this.loading = false;
                    this.fetchRequests();
                });
            }

            if ( 'paypal' === action ) {
                let ids = items.join(",");

                jQuery.post(ajaxurl, {
                    'dokan_withdraw_bulk': 'paypal',
                    'id': ids,
                    'action': 'withdraw_ajax_submission',
                    'nonce': dokan.nonce
                }, function(response, status, xhr) {
                    if ( 'html/csv' === xhr.getResponseHeader('Content-type') ) {
                        var filename = "";
                        var disposition = xhr.getResponseHeader('Content-Disposition');

                        if (disposition && disposition.indexOf('attachment') !== -1) {
                            var filenameRegex = /filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/;
                            var matches = filenameRegex.exec(disposition);

                            if (matches != null && matches[1]) {
                                filename = matches[1].replace(/['"]/g, '');
                            }
                        }

                        var type = xhr.getResponseHeader('Content-Type');

                        var blob = typeof File === 'function' ? new File([response], filename, { type: type }) : new Blob([response], { type: type });
                        if (typeof window.navigator.msSaveBlob !== 'undefined') {
                            // IE workaround for "HTML7007: One or more blob URLs were revoked by closing the blob for which they were created. These URLs will no longer resolve as the data backing the URL has been freed."
                            window.navigator.msSaveBlob(blob, filename);
                        } else {
                            var URL = window.URL || window.webkitURL;
                            var downloadUrl = URL.createObjectURL(blob);

                            if (filename) {
                                // use HTML5 a[download] attribute to specify filename
                                var a = document.createElement("a");
                                // safari doesn't support this yet
                                if (typeof a.download === 'undefined') {
                                    window.location = downloadUrl;
                                } else {
                                    a.href = downloadUrl;
                                    a.download = filename;
                                    document.body.appendChild(a);
                                    a.click();
                                }
                            } else {
                                window.location = downloadUrl;
                            }

                            setTimeout(function () {
                                URL.revokeObjectURL(downloadUrl);
                            }, 100); // cleanup
                        }
                    }

                    if ( response ) {
                        alert( self.massPayment );
                        return;
                    }
                });
            }
        },

        openNoteModal(note, id) {
            this.showModal = true;
            this.editing = {
                id: id,
                note: note
            }
        },

        updateNote() {
            this.showModal = false;
            this.loading = true;

            dokan.api.put('/withdraw/' + this.editing.id, {
                note: this.editing.note
            })
            .done((response) => {
                this.loading = false;

                this.updateItem(this.editing.id, response);
                this.editing = {
                    id: null,
                    note: null
                }
            });
        },

        clearSelection(element) {
            $(element).val(null).trigger('change');
        },
    }
};
</script>

<style lang="less">
.withdraw-requests {

    .dokan-modal {

        .modal-body {
            min-height: 130px;

            textarea {
                width: 100%;
            }
        }
    }

    .image {
        width: 10%;
    }

    .seller {
        width: 20%;
    }

    td.seller img {
        float: left;
        margin-right: 10px;
        margin-top: 1px;
        width: 24px;
        height: auto;
    }

    td.seller strong {
        display: block;
        margin-bottom: .2em;
        font-size: 14px;
    }

    td.actions,
    th.actions {
        width: 120px;
    }

    td.status {
        span {
            line-height: 2.5em;
            padding: 5px 8px;
            border-radius: 4px;
        }

        .approved {
            background: #c6e1c6;
            color: #5b841b;
        }

        .pending {
            background: #f8dda7;
            color: #94660c
        }

        .cancelled {
            background: #eba3a3;
            color: #761919
        }
    }

    .method_details_inner {
        p {
            margin-bottom: 2px;
        }
    }
}

@media only screen and (max-width: 600px) {
    .withdraw-requests {

        table td.seller, td.amount, td.actions {
            display: table-cell !important;
        }

        table th:not(.check-column):not(.seller):not(.amount):not(.actions) {
            display: none;
        }

        table td:not(.check-column):not(.seller):not(.amount):not(.actions) {
            display: none;
        }

        table th.column, table td.column {
            width: auto;
        }

        table td.column.actions .dashicons {
            width: 14px;
            height: 14px;
            font-size: 18px;
        }

        table td.seller {
            .row-actions {
                display: inline-block;
                span {
                    font-size: 11px;
                }
            }
        }
    }
}

@media only screen and (max-width: 376px) {
    .withdraw-requests {
        table td.seller {
            .row-actions {
                display: inline-block;
                span {
                    font-size: 9px;
                }
            }
        }
    }
}

@media only screen and (max-width: 320px) {
    .withdraw-requests {
        table td.column.actions .dashicons {
            width: 10px;
            height: 10px;
            font-size: 14px;
        }
    }
}
</style>
